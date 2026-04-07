---
title: "JDK11 HttpClientをリクエストごとにnewしていたら20万スレッドまで膨れた話"
emoji: "🧵"
type: "tech"
topics: ["Java", "JVM", "performance", "HttpClient", "JDK11"]
published: true
---

# JDK11 HttpClientをリクエストごとにnewしていたら20万スレッドまで膨れた話

外部 API 連携の新規実装を含む案件で、負荷テスト中に **GC Pause Time の右肩上がり** を観測しました。

レスポンスタイムは一見正常。しかし周期的にスパイクが発生し、**3秒近く** にまで跳ね上がるタイミングがありました。

原因は **`java.net.http.HttpClient` のスレッドリーク** でした。

-----

## 環境

|項目    |内容          |
|------|------------|
|JDK   |11          |
|AP サーバ|WebLogic 14c|
|負荷    |30 TPS      |

-----

## 発生した症状

負荷テスト中に次の症状が出ていました。

- GC Pause Time が **平均 50ms ずつ上昇傾向**
- 周期的なレスポンススパイクが **3秒** に到達
- レスポンスの中央値自体は正常

GC の Pause Time が右肩上がりという時点で「何かが溜まり続けている」のは明らかでした。

-----

## 調査の流れ

### まず疑ったこと：コネクションリーク

HTTP 通信を行っている箇所がある以上、まず疑うのは **コネクションリーク**（ソケットの枯渇）です。

`netstat` で確認しましたが、TCP コネクション数は一定の範囲内で推移しており、正常にクローズされていました。

```
# コネクションは正常に解放されている
$ netstat -an | grep ESTABLISHED | wc -l
→ 安定した数値で推移
```

コネクションリークではない。

### 次に確認：HeapDump

HeapDump を取得して解析しましたが、特にめぼしいオブジェクトの異常増殖は見つかりませんでした。

ヒープ上のオブジェクトサイズとしては目立たないが、何かが GC を圧迫している。

### JFR で決定的な証拠

**JFR（Java Flight Recorder）** を取得し、メモリサンプラーを確認したところ、決定的な情報が見つかりました。

**スレッド数が異常に増加していた。**

スレッドダンプを確認すると、以下のようなスレッドが大量に生存していました。

```
"HttpClient-1-SelectorManager" #xxx daemon
"HttpClient-1-Worker-0" #xxx daemon
"HttpClient-1-Worker-1" #xxx daemon
"HttpClient-2-SelectorManager" #xxx daemon
"HttpClient-2-Worker-0" #xxx daemon
...
"HttpClient-19998-SelectorManager" #xxx daemon
```

`HttpClient-N-SelectorManager` と `HttpClient-N-Worker-M` が際限なく増え続けていました。

最終的にスレッド数は **約 20万** まで膨れ上がっていました。

-----

## 原因

外部 API 通信用の Service クラスで、コンストラクタ内で `HttpClient` を毎回生成していました。

```java
// ❌ Before: コンストラクタで毎回 new
public class ExternalApiService {

    private final HttpClient httpClient;

    public ExternalApiService() {
        this.httpClient = HttpClient.newHttpClient();
    }

    public String call(String url) throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .build();
        HttpResponse<String> response = httpClient.send(
                request, HttpResponse.BodyHandlers.ofString());
        return response.body();
    }
}
```

この Service がリクエストのたびにインスタンス化されていたため、**リクエストごとに新しい `HttpClient` が生成**されていました。

### なぜスレッドが解放されないのか

`HttpClient.newHttpClient()` は内部で以下のリソースを生成します。

|内部リソース             |名前                            |役割                                              |
|-------------------|------------------------------|------------------------------------------------|
|**SelectorManager**|`HttpClient-N-SelectorManager`|NIO イベントループ。コネクション管理、タイムアウト監視を担当                |
|**Worker スレッドプール** |`HttpClient-N-Worker-M`       |`Executors.newCachedThreadPool()` で生成。上限なし      |
|**ConnectionPool** |—                             |TCP コネクションをキャッシュ。デフォルト keep-alive **1200秒（20分）**|

ここで重要なのは、JDK11 の `HttpClient` には **`close()` メソッドが存在しない** という点です。

リソースの解放は **GC に完全に依存した設計** になっています。内部的には以下の仕組みです。

1. ユーザーに返される `HttpClient` は実際には `HttpClientFacade`（ラッパー）
1. 内部の `HttpClientImpl` は `HttpClientFacade` への `WeakReference` を保持
1. `SelectorManager` スレッドが定期的に `WeakReference` をチェック
1. Facade が GC されると `WeakReference` がクリアされ、`SelectorManager` が停止

**問題は、`SelectorManager` スレッド自身が `HttpClientImpl` への強参照を保持している** ことです。

```
SelectorManager スレッド（GC Root）
  ↓ 強参照
HttpClientImpl
  ↓ 保持
ConnectionPool, SSLContext, Worker スレッドプール, バッファ ...
  ↓ WeakReference
HttpClientFacade（ユーザーが触るオブジェクト）
```

`SelectorManager` が生きている限り、`HttpClientImpl` 全体が GC Root から到達可能です。Facade が参照されなくなっても、`SelectorManager` の select ループが次にタイムアウトするまで（デフォルト **3秒**）は `WeakReference` のチェックが走りません。

しかし実際のところ、30 TPS でリクエストごとに新しい `HttpClient` が生成される状況では、**古い HttpClient の Facade が GC される前に次の HttpClient が生成される** ため、GC による回収が追いつかず、スレッドが際限なく溜まり続けます。

これが **スレッドリーク** の正体です。

### GC Pause Time が上がる理由

大量のスレッドが GC に与える影響は主に 2 つあります。

1. **Root Scanning の増大** — GC の Initial Mark / Remark（STW）フェーズでは、全スレッドのスタックを GC Root としてスキャンします。20万スレッドは膨大な Root Scanning コストになります
1. **Live Object の増大** — 各 `SelectorManager` が `HttpClientImpl` のオブジェクトグラフ全体を生存させるため、Concurrent Mark が走査すべき Live Object 量が増加し、Remark の処理量も増加します

### なぜ HeapDump で気づきにくいのか

`HttpClient` 自体の Retained Size は小さいため、HeapDump 上では Dominator Tree の上位に出にくい。しかし、抱えている **スレッド（OS リソース）** とそのスタック、そして間接的に保持される `ConnectionPool` のコネクション群が GC を圧迫します。

HeapDump はヒープ上のオブジェクトを可視化しますが、**スレッド数の異常** を直感的に示してはくれません。今回は JFR のスレッド情報が決定打になりました。

-----

## 修正（Step 1: static 化）

まず、`HttpClient` を `static` フィールドにして、インスタンス間で共有するようにしました。

```java
// ✅ After: static で共有
public class ExternalApiService {

    private static final HttpClient HTTP_CLIENT = HttpClient.newHttpClient();

    public String call(String url) throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .build();
        HttpResponse<String> response = HTTP_CLIENT.send(
                request, HttpResponse.BodyHandlers.ofString());
        return response.body();
    }
}
```

これにより、`HttpClient` は JVM 内で 1 つだけ生成され、`SelectorManager` も Worker プールも 1 セットに集約されます。

-----

## 修正結果

|指標           |Before       |After     |
|-------------|-------------|----------|
|スレッド数        |**約 200,000**|**200 未満**|
|GC Pause Time|平均 50ms 上昇傾向 |**安定**    |
|レスポンススパイク    |**3秒**       |**1秒**    |

-----

## 修正（Step 2: ライブラリ変更）

`static` 化でスレッドリークは解消しましたが、その後 **別の問題** が発生しました。

ロードバランサ配下のサーバの 1 台がダウンした後、復旧しても **そのサーバにリクエストが振り分けられない** という事象です。

原因は、`java.net.http.HttpClient` の内部コネクションプールにあります。デフォルトの keep-alive タイムアウトは **1200秒（20分）** と非常に長く、一度確立した TCP コネクションを長時間保持し続けます。L4 ロードバランサが送信元ポートを基に振り分けを行っている場合、コネクションが再利用される限り同じ送信元ポートが使われるため、ダウンしたサーバが復帰しても振り分け先が元に戻りませんでした。

この問題を受けて、最終的に **Apache HttpClient** に移行しました。

```java
// 最終形: Apache HttpClient
public class ExternalApiService {

    private static final CloseableHttpClient HTTP_CLIENT = HttpClients.createDefault();

    public String call(String url) throws Exception {
        HttpGet request = new HttpGet(url);
        request.setHeader("Connection", "close");
        try (CloseableHttpResponse response = HTTP_CLIENT.execute(request)) {
            return EntityUtils.toString(response.getEntity());
        }
    }
}
```

リクエストヘッダに `Connection: close` を付与することで、レスポンス受信後に TCP コネクションが確実にクローズされます。これにより、次のリクエストでは新しいコネクション（新しい送信元ポート）が使用されるため、LB が復旧サーバへ正しく振り分けできるようになりました。

Apache HttpClient はコネクション管理が明示的で、こうしたヘッダレベルの制御もシンプルに行えるため、LB 環境との相性問題が解消されました。

-----

## 補足：JDK 21 での改善

この問題の根本原因は **JDK11 の `HttpClient` に `close()` が存在しない** という設計にあります。

JDK 21 では [JDK-8304165](https://bugs.openjdk.org/browse/JDK-8304165) により `HttpClient` が `AutoCloseable` になり、`close()` / `shutdown()` / `shutdownNow()` が追加されました。また [JDK-8288746](https://bugs.openjdk.org/browse/JDK-8288746) により、JDK 20 以降では `Cleaner` ベースの即時リソース回収が導入されています。

keep-alive のデフォルト値も **1200秒 → 30秒** に短縮されました（[JDK-8297030](https://bugs.openjdk.org/browse/JDK-8297030)）。

つまり、今回のスレッドリークと LB 振り分け問題は、**JDK 21 であれば設計レベルで解消されている** 問題です。JDK 11 を使い続ける場合は、`static` 化に加えて以下のシステムプロパティによる緩和が推奨されます。

```
-Djdk.httpclient.keepalive.timeout=30
-Djdk.httpclient.connectionPoolSize=20
```

-----

## 教訓

### `HttpClient.newHttpClient()` を気軽に呼んではいけない

JDK11 の `java.net.http.HttpClient` は軽量なオブジェクトに見えますが、内部では以下のリソースを抱えています。

- **SelectorManager スレッド**（NIO イベントループ）
- **Worker スレッドプール**（`CachedThreadPool`、上限なし）
- **ConnectionPool**（デフォルト keep-alive 1200秒）

そして JDK11 には `close()` がないため、これらのリソース解放は GC 任せです。1 リクエスト 1 インスタンスの使い方は、GC が追いつかない速度でリソースを生成し続ける結果になります。

### 「コネクションは閉じている」≠「リソースは解放されている」

`netstat` が正常でも安心できません。今回のケースでは TCP コネクションは正しくクローズされていましたが、JVM 内部の SelectorManager スレッドは残り続けていました。

コネクションリークとスレッドリークは別物です。

### 調査の武器

|症状       |有効なツール                                                |
|---------|------------------------------------------------------|
|ヒープの異常   |HeapDump（jcmd / MAT）                                  |
|スレッドの異常  |**JFR**、スレッドダンプ（`HttpClient-N-SelectorManager` を grep）|
|コネクションの異常|netstat、tcpdump                                       |

今回のケースでは HeapDump だけでは原因にたどり着けず、**JFR のスレッド情報** が決定打になりました。

スレッドダンプでは `HttpClient-N-SelectorManager` の N の値が異常に大きい（= HttpClient インスタンスが大量に生成されている）ことが一目で分かります。

-----

## まとめ

|時系列 |内容                                                                                    |
|----|--------------------------------------------------------------------------------------|
|発覚  |負荷テストで GC Pause Time の右肩上がりを観測                                                        |
|調査  |netstat → 正常、HeapDump → めぼしい情報なし、JFR → `HttpClient-N-SelectorManager` が大量生存           |
|原因  |リクエストごとの `HttpClient.newHttpClient()` によるスレッドリーク。JDK11 には `close()` がなく GC 依存のリソース解放設計|
|修正  |`HttpClient` を `static` 化 → スレッド数 200,000 → 200 未満に                                   |
|追加対応|LB 振り分け問題（keep-alive 1200秒によるポート固定化）により Apache HttpClient + `Connection: close` に移行   |
|調査期間|約 1 週間                                                                                |

`java.net.http.HttpClient` は JDK11 で導入された標準 HTTP クライアントですが、ライフサイクル管理を誤ると深刻なリソースリークを引き起こします。

もし負荷テストで **GC Pause Time が右肩上がり** なのに **ヒープ使用量は正常** という症状が出たら、まずスレッドダンプを取って `HttpClient-N-SelectorManager` を探してみてください。
