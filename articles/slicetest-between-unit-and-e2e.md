---
title: "「ユニットテスト不要、E2Eに寄せればいい」論の間に置くテストフレームワーク slicetest を作った"
emoji: "🍰"
type: "tech"
topics: ["test", "testing", "vitest", "postgresql", "typescript"]
published: true
---

## はじめに

最近、テストの話題で「**ユニットテストはもう要らない。E2Eテストに寄せればいい**」という意見をよく見かけるようになりました。

それに対して「いや、ユニットテストこそ土台だ」という反論もあって、議論は広がり続けています。

どちらの言い分にも納得できるところがあります。ただ、両方を読んでいて感じたのは、

> **問題になっているのは、そのちょうど間の層なんじゃないか？**

ということでした。

そこで、ユニットテストとE2Eテストの**中間に位置するテスト**を書くためのフレームワーク **slicetest** を作ってみました。

- GitHub: https://github.com/revo1290/slicetest
- npm: https://www.npmjs.com/package/slicetest

この記事では、なぜ中間が欲しかったのか、slicetest で何ができるのかを紹介します。

## 両方の言い分を整理してみる

### 「ユニットテスト不要」派の言い分

ユニットテストは、DBや外部APIをモックに置き換えてテストするのが一般的です。すると、こういうことが起きます。

- モックが返すデータと、本物のDBが返すデータが食い違っていても気づけない
- SQLの書き間違いやマイグレーション漏れは、そもそもテストの対象外
- 外部APIに送るリクエストの形が間違っていても、モックは何も言わない
- リファクタリングするたびにモックの修正が必要になる

「結局、本番で壊れるのはモックで隠していた部分じゃないか」というのは、もっともな指摘です。

### 「E2E に寄せすぎるな」派の言い分

一方で、E2Eテストはブラウザを動かし、デプロイされた環境一式に対して実行します。

- 実行が遅いので、こまめに回せない
- タイミング依存で不安定（flaky）になりやすい
- 落ちたときに「どこが壊れたのか」の切り分けが難しい
- 外部サービスの異常系（タイムアウト、500エラーなど）を再現するのが難しい

「E2Eだけでは、変更のたびに安心して回せるテストにならない」というのも、これまたもっともです。

### 欲しいのは「本物」と「速さ」の両立

両方の言い分を並べると、欲しいものが見えてきます。

- DB・SQL・マイグレーションは**本物**で確かめたい（ユニットテストの弱点）
- でも、ブラウザやデプロイ済み環境までは要らない。**速く、決定的に**回したい（E2Eの弱点）
- 外部APIは自分たちが持っているものではないので、**そこだけ差し替えたい**

これを表にすると、こうなります。

| | ユニットテスト | **slicetest** | E2Eテスト |
|---|---|---|---|
| アプリ | 関数・クラス単位 | **本物のプロセス** | デプロイされた環境 |
| DB | モック | **本物の Postgres** | 本物 |
| 外部API | モック | **スタブサーバー** | 本物 or サンドボックス |
| ブラウザ | なし | **なし（HTTPで叩く）** | あり |
| 速さ・安定性 | ◎ | ○ | △ |
| 異常系の再現 | ○ | **◎** | △ |

アプリを「切り出して（slice）」テストするので、slicetest という名前にしました。

## slicetest でのテストはこうなる

投票アプリを例にします。「投票を作ると、DBに保存されて、Slackに通知される」という仕様をテストします。

```ts
import { expect } from "vitest";
import { scenario } from "slicetest";

scenario("投票を作るとDBに保存され、Slackに通知される", async ({ http, db, stub }) => {
  // 外部API（Slack）はスタブで差し替える
  stub("slack").on("POST", "/hook").reply(200, "ok");

  // 本物のアプリにHTTPリクエストを送る
  const res = await http.post("/polls", { title: "犬か猫か", a: "犬", b: "猫" });

  // レスポンス・DB・外部APIへの呼び出しを、1つのシナリオでまとめて確認する
  expect(res).toHaveStatus(201);
  await expect(db).toHaveRow("polls", { title: "犬か猫か" });
  expect(stub("slack")).toHaveReceived("POST", "/hook", {
    json: { text: "新しい投票: 犬か猫か（犬 vs 猫）" },
  });
});
```

ポイントは、**1つのシナリオで「入口・DB・出口」の3方向をまとめて確認できる**ことです。

- 入口：アプリへのHTTPリクエストとレスポンス
- DB：本物の Postgres に何が書き込まれたか
- 出口：外部APIにどんなリクエストが飛んだか

ブラウザは使いません。DBのモックもありません。アプリのコードにテスト用のフックを仕込む必要もありません。

## アプリ側に求めることは1つだけ

slicetest がアプリに求めるのは、**ポート番号・DBのURL・外部APIのベースURLを環境変数から読むこと**だけです。

設定は Vitest のプラグインとして書きます。

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
import { slicetest } from "slicetest/vitest";

export default defineConfig({
  plugins: [
    slicetest({
      app: {
        command: "python server.py", // 言語は問わない
        env: {
          PORT: "{{app.port}}",
          DATABASE_URL: "{{db.url}}",
          SLACK_WEBHOOK_URL: "{{stub.slack}}/hook",
        },
        ready: { path: "/health" },
      },
      db: {
        migrate: { atlas: { dir: "file://migrations" } }, // { sql: "schema.sql" } や { command: "npm run migrate" } も可
        seed: "seed.sql",
      },
      stubs: ["slack"],
    }),
  ],
  test: { include: ["scenarios/**/*.test.ts"] },
});
```

`{{stub.slack}}` の部分には、slicetest が立てたスタブサーバーのURLが入ります。アプリは「Slack の URL が環境変数で渡ってきた」としか認識しないので、本番と同じコードのままテストできます。

`app.command` はただのシェルコマンドなので、**アプリは Node でも Python でも Go でも構いません**。リポジトリの `examples/` には、Node（`node:http` + `pg`）と Python（`http.server` + `psycopg`）で同じAPIを実装したアプリがあり、**まったく同じシナリオファイルを両方に対して実行**しています。

## 仕組み：どうやって速さと独立性を両立しているか

中間層のテストで一番悩ましいのが、「本物のDBを使うと、テスト同士が干渉する・遅くなる」問題です。slicetest は次の流れで実行します。

1. **実行ごとに1回**：`postgres:17-alpine` を起動し、テンプレートDBにマイグレーションを適用する
2. **ワーカーごとに1回**：テンプレートDBをクローンして、ワーカー専用のDBを作る
3. **テストファイルごとに1回**：スタブサーバーとアプリを起動する
4. **シナリオの前に毎回**：テーブルを TRUNCATE して seed を入れ直し、スタブ・Cookie・リクエスト履歴をクリアする。前のシナリオでアプリがクラッシュしていたら再起動する
5. **シナリオの後に毎回**：アプリがクラッシュしていたり、登録していないスタブのルートを呼んでいたりしたら、シナリオを失敗にする

### DB のリセットは TRUNCATE 一発

シナリオ間のリセットは、次の SQL を1回実行するだけです。

```sql
TRUNCATE ... RESTART IDENTITY CASCADE
```

所要時間は約 1.5ms です。マイグレーションの管理テーブル（`atlas_schema_revisions`、`_prisma_migrations`、`alembic_version`、`django_migrations` など）や、PostGIS の `spatial_ref_sys` のような拡張機能のテーブルは対象外にしています。

「DBを DROP して作り直す」方式だと約 130ms かかるうえに、コネクションプールを持つアプリがコネクションを切られてクラッシュすることがありました。TRUNCATE 方式ならアプリはコネクションを保持したままなので、**ドライバや ORM を問わず動きます**。

`RESTART IDENTITY` を付けているので、シナリオごとに ID も 1 から振り直されます。「前のテストで作ったデータのせいで ID がずれて落ちる」といった事故も起きません。

## E2E では難しい「異常系」が書きやすい

中間層に置く大きなメリットの1つが、**外部APIの異常系が簡単に書ける**ことです。E2Eで「Slack が 500 を返したとき」「接続が切れたとき」を再現するのは大変ですが、スタブなら1行です。

```ts
scenario("Slack通知に失敗したら投票は作られない", async ({ http, db, stub }) => {
  stub("slack").on("POST", "/hook").reply(500);

  const res = await http.post("/polls", { title: "山か海か", a: "山", b: "海" });

  expect(res).toHaveStatus(502);
  await expect(db).toHaveRow("polls", { title: "山か海か" }, 0); // ロールバックされている
});

scenario("Slackに繋がらなくても投票は作られない", async ({ http, db, stub }) => {
  stub("slack").on("POST", "/hook").networkError(); // 接続を切る

  expect(await http.post("/polls", { title: "夏か冬か", a: "夏", b: "冬" })).toHaveStatus(502);
  await expect(db).not.toHaveRow("polls", { title: "夏か冬か" });
});
```

「外部APIの呼び出しに失敗したら、DBへの書き込みもロールバックされるか」は、**モックのユニットテストでは確かめにくく、E2Eでは再現しにくい**、まさに中間でしか見られない性質です。

他にも、リトライやタイムアウトの確認ができます。

```ts
stub("slack").on("POST", "/hook").once().reply(500);          // 1回目だけ失敗させて…
stub("slack").on("POST", "/hook").reply(200);                 // …2回目は成功（リトライの確認）
stub("pay").on("GET", "/status").replySequence([{ status: 503 }, { status: 200 }]);
stub("pay").on("POST", "/charge").delay(5_000).reply(200);    // アプリのタイムアウトを確認
```

また、**登録していないルートへの呼び出しは 501 を返し、シナリオを失敗させます**。「知らないうちに外部APIを叩いていた」という見落としも拾えます。

## 落ちたときに「何が起きたか」が分かる

E2Eの辛さの1つが、落ちたときの原因調査です。slicetest では、シナリオが失敗すると **そのシナリオの間に起きたこと** を Vitest のエラーと並べて表示します。

```
--- slicetest ---
stub calls with no matching route:
  mail: POST /send
    registered on mail: POST /other

requests to the app:
  POST /signup → 500 (14ms)  {"error":"internal"}

database changes during this scenario:
  users: 1 inserted
    + {"id":1,"email":"a@example.com","verified":false}
  audit_log: 1 updated
    ~ id=7  status: "pending" → "failed"

app output during this scenario:
TypeError: Cannot read properties of undefined (reading 'email')
-----------------
```

- アプリに送ったリクエストと、そのレスポンス
- マッチしなかった外部APIの呼び出し
- DBに書き込まれた差分
- **そのシナリオの間だけ**のアプリのログ

が一度に見えるので、ログ全体を掘り返す必要がありません。

:::message
上の出力のうち「database changes during this scenario」の部分は、次に紹介する `db.changes()` と一緒に入る機能で、npm で公開中の 0.1.0 にはまだ含まれていません。
:::

## 次のリリースで入る `db.changes()`：アプリが書いたものを丸ごと検証する

:::message alert
`db.changes()` は GitHub の main には入っていますが、**npm で公開中の 0.1.0 にはまだ含まれていません**。次のリリースで入る予定です。
:::

「このAPIを叩いたら、どのテーブルに何が書かれたか」を、テーブルを1つずつ SELECT して確認するのは面倒ですし、確認し忘れたテーブルへの書き込みは見逃してしまいます。

`db.changes()` は、シナリオ中にアプリが書き込んだ内容を**主キーで突き合わせた差分**として返します。

```ts
await db.insert("users", { email: "a@example.com" });
await db.checkpoint(); // ここまでのテスト側の準備は差分に含めない

await http.post("/users/1/verify");

expect(await db.changes()).toEqual({
  users: { inserted: [], deleted: [], updated: [expect.objectContaining({ changed: ["verified"] })] },
  audit_log: { inserted: [expect.objectContaining({ action: "verify" })], updated: [], deleted: [] },
});
```

`toEqual` で比較するので、**列挙していないテーブルに書き込みがあればテストが落ちます**。「想定外の副作用がないこと」まで確認できるわけです。

## JavaScript を書かないチーム向け：YAML シナリオ

アプリが Python や Go で書かれているチームだと、「テストのために TypeScript を書くのはちょっと…」となることもあると思います。そこで、同じことを YAML でも書けるようにしました。

```yaml
scenarios:
  - name: 投票を作るとDBに保存され、Slackに通知される
    steps:
      - stub: slack
        on: POST /hook
        reply: { status: 200, body: ok }

      - request: POST /polls
        json: { title: 犬か猫か, a: 犬, b: 猫 }
        expect:
          status: 201
          json: { id: { $type: number } }
        capture: { pollId: json.id }

      - db: polls
        where: { title: 犬か猫か }
        expect:
          rows: [{ id: "{{pollId}}", option_a: 犬 }]

      - received: slack
        call: POST /hook
        times: 1
```

設定を `slicetest.config.yaml` に書けば、`package.json` も Vitest の設定も不要で、CLI だけで実行できます（Node 自体は必要です）。

```sh
npx slicetest                 # 配下の *.scenario.yaml をすべて実行
npx slicetest polls -t 投票   # ファイル名・シナリオ名で絞り込み
```

YAML のキーを打ち間違えると、実行前に `polls.scenario.yaml:12: unknown key "stauts" in expect` のようにファイル名と行番号付きで教えてくれます。JSON Schema も同梱しているので、エディタで補完も効きます。

## 使ってみるには

```sh
npm i -D slicetest vitest
```

Docker か Podman が必要です（Podman マシンも自動で見つけます）。既存の Postgres を使いたい場合は、`db.url` か環境変数 `SLICETEST_DATABASE_URL` で指定できます。CI のサービスコンテナを使う場合はこちらが便利です。

macOS・Linux・Windows で動作し、CI では Linux と Windows の両方で回しています。

## 向いていないこと・現状の制約

万能ではないので、制約も書いておきます。

- **DB は Postgres のみ**です（MySQL 対応は予定しています）
- **HTTP で叩けるアプリ**が対象です。画面の見た目や操作感の確認はできないので、そこは E2E の出番です
- 1つのテストファイル内のシナリオは、アプリとDBを共有するため**直列に実行**されます（ファイル単位では並列）
- 純粋なロジック（計算、バリデーション、変換処理など）は、ユニットテストのほうが速くて的確です
- まだ**初期段階**のツールです

## おわりに

「ユニットテストか、E2Eか」という議論は、どちらかに寄せるかの二択として語られがちです。でも実際には、

- 純粋なロジックは**ユニットテスト**で速く確かめる
- DB・SQL・外部APIとのつなぎ目は**中間のテスト**で本物を使って確かめる
- 画面を含むユーザー体験は**E2E**で少数を確かめる

と、層ごとに役割を分けるのが現実的なんじゃないかと思っています。

「ユニットテスト不要論」が出てくる背景には、モックだらけのテストが本当に守りたいものを守れていなかった、という実感があるはずです。そして「E2Eに寄せすぎるな」の背景には、遅くて不安定なテストに疲れた、という実感があるはずです。

slicetest は、その両方の実感から「本物を使いつつ、速く、壊れた場所が分かる」テストを書けるようにする試みです。興味があれば触ってみてもらえると嬉しいです。Issue や PR も歓迎しています。

- GitHub: https://github.com/revo1290/slicetest
- npm: https://www.npmjs.com/package/slicetest
