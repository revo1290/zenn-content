---
title: "Chrome の Modern Web Guidance を Claude に入れると、書くコードはどう変わるのか（72 実装で検証）"
emoji: "🌐"
type: "tech"
topics: ["claudecode", "frontend", "css", "chrome", "ai"]
published: true
---

## はじめに

Google Chrome チームが公開している **Modern Web Guidance**（以下 MWG）は、AI コーディングエージェントに「今のブラウザで使える、今どきの書き方」を教えるためのスキルです。Claude Code ならプラグインとして入れるだけで、HTML / CSS / クライアント JS の作業時にガイドを検索して参照してくれるようになります。

気になったのは、**実際に入れると Claude が書くコードは何がどう変わるのか**です。そこで、同じ依頼を MWG「なし」と「あり」で実装させて比較しました。

:::message
**検証の概要**

- 検証日: 2026-11-18〜19
- 対象: Modern Web Guidance v1.0.1（skill-version `2026_11_10-47b4c80e`）
- 実装したモデル: Claude Opus 5.5 / Claude Sonnet 5.5（各モデルで全条件を実行）
- 規模: お題 16 本 ×（なし / あり）＋派生条件 4 本 ＝ 36 実装 × 2 モデル ＝ **計 72 実装**
- 計測: 静的解析（使った API、レガシーパターン、行数）、MWG 付属の `legacy-scan`、Chrome 153 での動作テストとスクリーンショット
:::

特に断りがない限り、記事前半の数値やスクリーンショットは Opus 5.5 の結果です。Sonnet 5.5 との比較は後半の「Opus 5.5 と Sonnet 5.5 の比較」にまとめています。

:::message
**この記事の位置づけ**

本記事は、個人が手元で行った簡単な検証の記録です。お題やテストの観点は自分で決めたもので、各条件 1 回ずつしか実行していないため、統計的な精度はありません。MWG や各モデルの精度・善し悪しを評価したり判断したりするものではなく、「この条件で試したら、こういう違いが出た」という観察結果として読んでいただければと思います。
:::

## 結果のまとめ

先に、今回の検証で見えたことをまとめます。どれも今回の条件で観察できたことで、一般的な傾向を示すものではありません。

1. **一番大きい差は JavaScript の量でした。** MWG ありでは、同じ要件を CSS / HTML の標準機能で書く割合が増え、JS 行数は合計で 4,376 → 2,740 行（**−37%**）になりました。16 題中 14 題で JS が減っています。極端な例では、カスタムセレクトが 429 → 12 行（`appearance: base-select`）、ツールチップとメニューが 625 → 300 行（Popover＋Anchor Positioning）、FAQ アコーディオンが 136 → 10 行です。
2. **使うモダン機能の数はおよそ 2 倍になりました。** 1 実装あたり平均 4.6 → 8.6 種類です。`closedby`、`interestfor`、`popover="hint"`、`scroll-state` クエリ、`animation-timeline`、`setHTML()`、`Temporal`、`light-dark()` など、MWG なしでは一度も使わなかった機能が、ありでは普通に出てきます。
3. **見た目と基本動作の差はほとんどありませんでした。** Esc・背景クリックで閉じる、排他開閉、XSS が発火しない、入力中はエラーを出さない、といった動作テストは、なしでも全部通っています。今回の範囲では、Opus 5.5 は MWG なしでもかなりモダンで丁寧に書いていました（`<dialog>`、`<details name>`、`interpolate-size`、Web Worker、Trusted Types などは自力で使っています）。差は品質の底上げというより、**実装の方針（JS で作るか、プラットフォームに任せるか）** に出たように見えます。
4. **MWG ありのほうがうまくいかなかったケースもありました。**
   - T13 の「あり」版は、Temporal がネイティブにある Chrome では実行時エラーになり、カウントダウンが動きませんでした。エージェントは Node＋ポリフィルでしか検証しておらず、ネイティブ経路のバグ（変数を初期化前に参照）を見逃していました。
   - T03 の「あり」と「limited」は外部 JS を ES モジュールで読み込んでおり、file:// で開くと JS が読み込まれません（依頼は「ブラウザでそのまま開ける」こと）。
   - 未対応ブラウザ向けに、**ポリフィルを CDN から読み込む実装が 4 件**出ました（なしでは 0 件）。うち 2 件は `@latest` でバージョン固定なしです。
5. **運用系の機能は、試した範囲では想定どおりに動きました。**
   - `.mwgrc` で `baseline: "limited"` にすると、フォールバックがきれいに消えて、JS がさらに 1/2〜1/4 になりました（T01: 115 → 30 行）。
   - 実験的な機能のゲートも設計どおり働きました。opt-in していない状態では要約 API（Summarizer）を採用せず、標準 JS の要約に切り替えています。一方、MWG なしの Claude は記憶だけを頼りに、ゲートなしで Summarizer を組み込んでいました。
6. **コスト**: 1 実装あたりのトークン消費は平均 +24%、所要時間は中央値で +38%（134 秒 → 185 秒）でした。
7. **気になった点**
   - CLI は既定で、**検索クエリ文字列とガイド ID を Google に送るテレメトリ**を持っています。止めるには `DISABLE_TELEMETRY=1` が必要です。
   - `legacy-scan` が見るのはガイド全体の約 2 割（38 ガイド分）だけです。仕込んだ 10 種類以上の古いパターンのうち、検出できたのは 3 件でした。
8. **Sonnet 5.5 のほうが、MWG あり / なしの差が大きく出ました。**
   - 今回の Sonnet は、MWG なしだと保守的で古めの書き方に寄っていました。使ったモダン機能は平均 2.6 種でしたが、MWG ありでは平均 6.5 種に増え、**MWG なしの Opus（4.6 種）を上回りました**。
   - 一方で JS は減りませんでした（55.4KB → 54.8KB）。Sonnet の場合、差は「コード削減」ではなく「新しい機能＋フォールバックの追加」として現れました。
   - **T13 は Opus・Sonnet とも「あり」だけが壊れました。**

## Modern Web Guidance とは

検証の前に、MWG の中身を調べた結果です。

| 項目 | 内容 |
|---|---|
| 提供 | GoogleChrome/modern-web-guidance（Apache-2.0）。README によると Chrome・Edge・Firefox・MDN の各チームが共同で管理している |
| 配布形態 | Claude Code プラグイン（`claude plugin install modern-web-guidance@claude-plugins-official`）、`npx modern-web-guidance@latest install`、Gemini CLI、Codex、Copilot CLI など多数 |
| 常駐する部分 | スキル一覧に載る description（約 200 トークン）だけ。description には「HTML/CSS とクライアント JS の作業では **MANDATORY で最初に実行**」と書かれている |
| 発火後に読む部分 | `SKILL.md`（約 9.8KB）。手順は search → retrieve → ガイドとの突き合わせ → 回答末尾に `### Modern Web Guidance` で出典を明記 |
| ガイド | **209 本**（ui-behaviors 40、performance 26、ui-components 22、visual-design 22、forms 19、css 18、security 17 など）。1 本あたり平均 2,334 トークン（中央値 1,769、最大は accessibility の 13,341）。全部で約 49 万トークン |
| 検索 | 埋め込みベクトルによる**オフライン検索**。API キーも外部の検索サービスも不要 |
| 互換性の扱い | 既定は「Baseline Widely available はフォールバックなし、それ以外は各ガイドのフォールバックを必須にする」。`.mwgrc` の `baseline`（`widely` / `newly` / `limited` / `2024` のような年指定）で切り替えられ、retrieve の結果の先頭に方針バナーが付く |
| ゲート | Origin Trial 系 4 ガイド（WebMCP など）と、1 エンジンしか実装していない機能（volatile）を使う 14 ガイド（Summarizer、Translator、Prompt API、HTML-in-canvas など）は、`.mwgrc` で opt-in しないと中身が返らない。「エージェント自身で opt-in コマンドを実行するな」「ユーザーに確認せよ」とも明記されている |
| legacy-scan | 既存コードから古いパターンを静的に検出するコマンド。対象は 38 ガイドだけで、本人も「コーパスの約 20%」と書いている |
| テレメトリ | **既定でオン**。検索クエリ、取得したガイド ID、OS、エージェント名、レイテンシを Google の Clearcut に送る。`DISABLE_TELEMETRY=1` で止まる |

## 検証方法

### 条件

| 条件 | 内容 |
|---|---|
| **なし (off)** | 依頼文だけを渡す |
| **あり (on)** | 依頼文に加えて、Claude Code のスキル一覧と同じ形で MWG の description を提示する。使うかどうかはモデルに任せる。CLI はローカルに入れた v1.0.1 をラッパー経由で呼び、全呼び出しを記録した（テレメトリはオフ） |
| **あり + baseline=limited** | 作業ディレクトリに `.mwgrc {"baseline":"limited"}` を置く（T01・T03・T05） |
| **あり + volatile opt-in** | `.mwgrc {"allowVolatileFeatures": true}` を置く（T15） |

共通ルールは次のとおりです。

- Web 検索は禁止
- 作業ディレクトリの外は読まない
- 依頼者には質問できない
- ブラウザでの目視確認はしない
- 各実装は独立したエージェントが担当し、ほかの条件の存在は知らせない

### お題（16 本）

| ID | 観点 | 種別 | 依頼（要約） |
|---|---|---|---|
| T01 | UI/CSS | 新規 | 削除確認モーダル。背景クリック・Esc で閉じ、フェード＋スケールで開閉 |
| T02 | UI/CSS | 新規 | 置き場所の幅で縦横が変わる商品カード。画像なしは 1 カラム |
| T03 | UI/CSS | 新規 | ヘルプのツールチップとアクションメニュー。画面端で反転し、矢印の向きも追従 |
| T04 | UI/CSS | 新規 | ブランドカラーの国旗付きカスタムセレクト |
| T05 | アニメ/スクロール | 新規 | FAQ アコーディオン（排他開閉、高さアニメ、閉じた中身もページ内検索でヒット） |
| T06 | アニメ/スクロール | 新規 | 読了プログレスバーと、スクロールで画像がフェードイン |
| T07 | アニメ/スクロール | 新規 | 貼り付いた時だけ変化する固定ヘッダーと「トップに戻る」ボタン |
| T08 | セキュリティ | 新規 | ユーザー入力 HTML のプレビュー（XSS を確実に防ぐ） |
| T09 | 性能 | 既存改修 | 用意したページの LCP / INP 改善 |
| T10 | フォーム/a11y | 新規 | 会員登録フォーム（バリデーションのタイミング、オートフィル） |
| T11 | 性能 | 新規 | 1 万行 CSV の集計で画面を固まらせない |
| T12 | 既存改修 | 既存改修 | jQuery 製の明細ページ（古いパターンを 10 種類以上仕込んだもの）を全面リファクタ |
| T13 | 最新機能 | 新規 | 3 都市のタイムゾーン表示とカウントダウン（夏時間対応） |
| T14 | 最新機能 | 新規 | 別 HTML 間でサムネイルがヒーロー画像に変形するページ遷移 |
| T15 | 最新機能 | 新規 | 外部 API なしでブラウザ内要約 |
| T16 | UI/CSS | 新規 | ライト / ダーク / システムの 3 択テーマ切替（フォーム部品・スクロールバーも追従） |

### 計測

- **静的解析**: モダン機能 59 種、レガシーパターン 21 種を正規表現で検出。JS・CSS の行数（空行とコメント行を除く）も数えた
- **legacy-scan**: MWG 付属の検出器を全成果物に実行
- **動作テスト**: Chrome for Testing 153（ヘッドレス）で実際に操作した
- **スクリーンショット**: 初期表示と操作後の 2 枚を撮り、条件別に横に並べた

## 全体結果

| 指標 | なし | あり |
|---|---|---|
| JS 行数（16 題合計） | 4,376 | **2,740（−37%）** |
| CSS 行数（16 題合計） | 3,364 | 3,462 |
| モダン機能数（1 実装の平均） | 4.6 | **8.6** |
| legacy-scan の指摘 | 11 | **0** |
| スキルの発火率 | — | **20 / 20**（派生条件を含む全実装で search と retrieve を実行） |
| 回答末尾の出典明記（`### Modern Web Guidance`） | — | 20 / 20 |
| 検索回数 / 取得ガイド数（平均） | — | 3.3 / 4.5 |
| トークン消費（参考、14 ペアの平均） | 90.4k | 112.1k（**+24%**） |
| 所要時間（14 ペアの中央値） | 134 秒 | 185 秒（**+38%**） |
| 動作テストの不合格 | 0 | **1**（T13 の実行時エラー）＋部分的に 1（T03：file:// でモジュールの JS が読み込まれない） |
| 外部 CDN 依存 | 0 件 | **4 件**（T03・T12 は `@latest`） |

:::details お題ごとの全体結果（Opus 5.5）

| ID | 観点 | 種別 | JS 行（なし→あり） | CSS 行 | モダン機能数 | legacy-scan 指摘 | 検索 / 取得ガイド数 |
|---|---|---|---|---|---|---|---|
| T01 | UI/CSS | 新規 | 145 → 115 | 195 → 197 | 7 → 8 | 1 → 0 | 3/6 |
| T02 | UI/CSS | 新規 | 0 → 8 | 177 → 238 | 4 → 5 | 0 → 0 | 2/3 |
| T03 | UI/CSS | 新規 | **625 → 300** | 430 → 409 | **4 → 15** | 0 → 0 | 3/4 |
| T04 | UI/CSS | 新規 | **429 → 12** | 238 → 162 | 3 → 7 | 1 → 0 | 2/2 |
| T05 | アニメ | 新規 | **136 → 10** | 148 → 112 | 9 → 10 | 0 → 0 | 3/3 |
| T06 | アニメ | 新規 | 96 → 43 | 181 → 148 | 3 → 8 | 1 → 0 | 2/2 |
| T07 | アニメ | 新規 | 72 → 33 | 228 → 222 | 3 → 4 | 0 → 0 | 2/2 |
| T08 | セキュリティ | 新規 | 189 → 130 | 111 → 115 | 3 → 5 | 1 → 0 | 2/3 |
| T09 | 性能 | 改修 | 43 → 40 | 6 → 20 | 3 → 4 | 0 → 0 | 4/5 |
| T10 | フォーム | 新規 | 259 → 201 | 260 → 209 | 4 → 4 | 2 → 0 | 3/6 |
| T11 | 性能 | 新規 | 564 → 494 | 145 → 186 | 4 → 8 | 0 → 0 | 3/4 |
| T12 | 既存改修 | 改修 | 84 → **186** | 133 → 223 | **6 → 24** | **4 → 0** | **10/14** |
| T13 | 最新機能 | 新規 | 436 → 377 | 209 → 199 | 3 → 8 | 0 → 0 | 4/4 |
| T14 | 最新機能 | 新規 | **491 → 187** | 294 → 218 | 7 → 9 | 0 → 0 | 1/3 |
| T15 | 最新機能 | 新規 | 675 → 558 | 165 → 367 | 4 → 11 | 1 → 0 | 6/5 |
| T16 | UI/CSS | 新規 | 132 → 46 | 444 → 437 | 6 → 8 | 0 → 0 | 3/6 |
| **合計 / 平均** | | | **4,376 → 2,740** | 3,364 → 3,462 | **4.6 → 8.6** | **11 → 0** | 平均 3.3 / 4.5 |

:::

## 観点別に見た差分

ここからはお題ごとに、なしとありで何が違ったかを見ていきます。スクリーンショットは左から「MWG なし」「MWG あり」（派生条件があるお題は 3 枚目に派生条件）の順に並べています。

### UI・レイアウト・CSS

#### T01 削除確認モーダル

![](/images/modern-web-guidance-ab-test/opus/T01_2.png)
*T01 モーダルを開いた状態。左から なし / あり / あり + baseline=limited*

- どちらも `<dialog>` と `showModal()` を使い、初期フォーカスはキャンセルボタンでした。動作テスト（開く・Esc・背景クリック）は、なしもありも全部通っています。
- **なし**: 背景クリックは自前の座標判定です。閉じるアニメは `transitionend` を待ってから `close()` する昔ながらの方式で、Esc も keydown で横取りしています（JS 145 行）。
- **あり**: `closedby="any"`、`@starting-style`、`transition-behavior: allow-discrete`、`overlay` を使い、CSS だけで開閉アニメを付けています。Safari 用の `closedby` 代替や、Firefox が「`allow-discrete` を解釈するのに実際は遷移しない」問題の検出も、ガイドから取り込んでいました（JS 115 行）。
- **baseline=limited**: フォールバックがすべて消え、`command="show-modal"` で開く完全な宣言型になりました（JS 30 行）。

#### T03 ツールチップ＋アクションメニュー

![](/images/modern-web-guidance-ab-test/opus/T03_2.png)
*T03 ツールチップを表示した状態。左から なし / あり / あり + baseline=limited*

- **なし**: Popover API はトップレイヤー表示にだけ使い、配置は `getBoundingClientRect` による自前の配置エンジンでした。メニュー操作は WAI-ARIA の作法に沿って JS で実装しています（JS 625 行）。
- **あり**: `popover="hint"` と `interestfor`、`commandfor`、`focusgroup="menu"`、Anchor Positioning（`position-try-fallbacks`）、`@container anchored(fallback: …)` で矢印の向きを切り替えています。モダン機能は 4 → 15 種に増え、JS は半分になりました。ただし未対応ブラウザ用に、**ポリフィル 2 本を unpkg から `@latest` で動的に読み込んでいます**。
- **baseline=limited**: ポリフィルと座標計算のフォールバックが消えて JS 156 行になりました。その代わり「今まともに動くのは最新の Chrome / Edge だけ」とエージェント自身が明記しています。

:::message alert
T03 の「あり」と「limited」は外部 JS を `<script type="module" src>` で読み込んでいるため、**file:// で開くと JS が読み込まれません**。Chrome では HTML の宣言だけでツールチップとメニューは開きますが、ARIA の補完と他ブラウザ向けのフォールバックは効きません。
:::

#### T04 カスタムセレクト

![](/images/modern-web-guidance-ab-test/opus/T04_2.png)
*T04 セレクトを開いた状態。左が なし、右が あり*

- **なし**: ARIA の Select-Only Combobox を JS でフル実装しました（JS 429 行）。`base-select` は「対応ブラウザが限られる」として意図的に見送っています。
- **あり**: ネイティブの `<select>` に `appearance: base-select`、`::picker(select)`、`<selectedcontent>` を組み合わせました（JS 12 行）。見た目はほぼ同じで、未対応ブラウザでは標準のセレクトに戻ります（国旗は出ません）。

**今回いちばん差が大きかったお題**です。どちらがよいかは、サポート対象のブラウザ次第になります。

#### T02 商品カード／T16 テーマ切替

- **T02** はどちらも Container Queries と `:has()` を使い、差は小さめでした。なしは `@supports not` によるフォールバックを付け、ありは「Widely available だから不要」として付けていません。**フォールバックの要否を Baseline に基づいて判断する**のが、MWG ありの特徴としてよく出ています。
- **T16** は、なしが `light-dark()` を使わず配色を 2 か所に重複定義していました（JS 132 行）。ありは `light-dark()` と、meta の `color-scheme` を `:root:has()` で CSS に反映する方式を使っています（JS 46 行）。ガイドにある「meta を書き換えるだけでは CSS が勝つので何も切り替わらない」という落とし穴を、明示的に避けていました。

:::details T01〜T04・T16 のその他のスクリーンショット

![](/images/modern-web-guidance-ab-test/opus/T01_1.png)
*T01 初期表示*

![](/images/modern-web-guidance-ab-test/opus/T02_1.png)
*T02 初期表示*

![](/images/modern-web-guidance-ab-test/opus/T03_1.png)
*T03 初期表示*

![](/images/modern-web-guidance-ab-test/opus/T04_1.png)
*T04 初期表示*

![](/images/modern-web-guidance-ab-test/opus/T16_1.png)
*T16 初期表示*

![](/images/modern-web-guidance-ab-test/opus/T16_2.png)
*T16 操作後*

:::

### アニメーション・スクロール

#### T05 FAQ アコーディオン

![](/images/modern-web-guidance-ab-test/opus/T05_2.png)
*T05 項目を開いた状態。左から なし / あり / あり + baseline=limited*

どちらも `<details name>`、`::details-content`、`interpolate-size` を使っていて、**なしでも最新のパターンを知っていました**。差はフォールバックの書き方だけで、なしは Web Animations API による JS 補完（136 行）、ありは CSS の `grid-template-rows: 0fr / 1fr`（JS 10 行）です。limited では JS 0 行になりました。

#### T06 プログレスバーとフェードイン

![](/images/modern-web-guidance-ab-test/opus/T06_2.png)
*T06 スクロール後。左が なし、右が あり*

- **なし**: scroll イベント＋`requestAnimationFrame`、IntersectionObserver で実装しました。
- **あり**: `animation-timeline: scroll()` / `view()` を使っています。部分対応のブラウザを除外する `@supports (animation-range: entry)` も、ガイドの必須指示どおり入っていました。JS は未対応ブラウザ用だけです（96 → 43 行）。

#### T07 固定ヘッダー

![](/images/modern-web-guidance-ab-test/opus/T07_2.png)
*T07 スクロール後。左が なし、右が あり*

- **なし**: 目印になる要素＋IntersectionObserver で実装しました。
- **あり**: `container-type: scroll-state` と `@container scroll-state(stuck: top)` を使っています。さらに、**ガイドのフォールバックコードをそのまま使うと初期表示で誤判定するバグ**をエージェントが見つけて修正していました。ガイドを鵜呑みにせず、検証している例です。

:::details T05〜T07 のその他のスクリーンショット

![](/images/modern-web-guidance-ab-test/opus/T05_1.png)
*T05 初期表示*

![](/images/modern-web-guidance-ab-test/opus/T06_1.png)
*T06 初期表示*

![](/images/modern-web-guidance-ab-test/opus/T07_1.png)
*T07 初期表示*

:::

### 性能・a11y・セキュリティ

#### T08 HTML プレビュー（XSS 対策）

どちらも動作テストで XSS は発火せず、`javascript:` リンクも残りませんでした。

- **なし**: DOMParser と許可リストで DOM を組み直し、ハッシュ方式の CSP と Trusted Types を強制しています。外部依存はありません。
- **あり**: `setHTML()`（Sanitizer API）をメインに使い、未対応ブラウザ向けに **DOMPurify を CDN から SRI 付きで読み込んでいます**。CSP は `'strict-dynamic'` 付きです。

堅牢さはほぼ同等で、ありは外部依存が 1 つ増えました。

#### T09 LCP / INP の改修

打ち手はほぼ同じでした（preload＋`fetchpriority`、`defer`、`scheduler.yield` による分割、`content-visibility`）。ありは「`fetchpriority=low` を lazy 画像に付けない」「`contain-intrinsic-size` を必ず組み合わせる」といった細かい規則を守っています。性能まわりは **Claude 単体でもかなり知っている**ようで、差は細部に限られました。

なお、この環境では元のページでも long task が出なかったため、性能差は数値化できていません。

#### T10 会員登録フォーム

どちらも「入力中はエラーを出さず、欄を離れたら出す」挙動を動作テストで確認できました。

- **なし**: JS のクラス制御で実現しています（JS 259 行）。
- **あり**: `:user-invalid` を使いました（JS 201 行）。ガイドに従ってメール欄を `autocomplete="username"` にしていますが、住所録のメールが候補に出なくなる副作用があります。エージェント自身もそれを報告していました。

#### T11 CSV 集計

どちらも Web Worker（Blob URL）を使い、`scheduler.yield` による分割実行をフォールバックにしていました。**ほぼ同じ設計**です。

:::details T08〜T11 のスクリーンショット

![](/images/modern-web-guidance-ab-test/opus/T08_1.png)
*T08 初期表示*

![](/images/modern-web-guidance-ab-test/opus/T08_2.png)
*T08 操作後*

![](/images/modern-web-guidance-ab-test/opus/T09_1.png)
*T09 初期表示*

![](/images/modern-web-guidance-ab-test/opus/T10_1.png)
*T10 初期表示*

![](/images/modern-web-guidance-ab-test/opus/T10_2.png)
*T10 操作後*

![](/images/modern-web-guidance-ab-test/opus/T11_1.png)
*T11 初期表示*

![](/images/modern-web-guidance-ab-test/opus/T11_2.png)
*T11 操作後*

:::

### 既存コードの改修（T12：jQuery 製の明細ページ）

古いパターンを 10 種類以上仕込んだ jQuery 製のページを、全面的にリファクタしてもらったお題です。

![](/images/modern-web-guidance-ab-test/opus/T12_1.png)
*T12 初期表示。左（なし）はメモ欄の HTML をすべて文字列として表示し、右（あり）は `<b>` だけを残して描画している*

| 項目 | なし | あり |
|---|---|---|
| 取得したガイド | — | 14 本（検索 10 回、legacy-scan 2 回） |
| モーダル | `<dialog>` | `<dialog>`＋`closedby` |
| ツールチップ | 位置ズレを JS で修正 | `popover="hint"`＋`interestfor`＋Anchor Positioning |
| FAQ | `<details>` | `<details>`＋`::details-content` のアニメ |
| ヘッダー縮小・トップへ戻る | scroll イベント（jQuery を外しただけ） | IntersectionObserver |
| メモ欄の XSS | `textContent`（`<b>` も文字列で表示） | `setHTML` で `<b>` だけ残す |
| テーマ | クラス切替 | `color-scheme`＋`light-dark()`（**初期値が OS 追従に変わった**） |
| 狭い画面の表 | そのまま | `@container`＋セルごとの列名 |
| legacy-scan の指摘 | **4 件**（dark-mode-toggle、dialog など） | **0 件** |
| JS / CSS 行 | 84 / 133 | 186 / 223 |
| 外部依存 | なし | ポリフィル 2 本を unpkg から `@latest` で読み込む |

- **あり**のほうが網羅的にモダン化しました（モダン機能 6 → 24 種）。ただし**挙動の変更**（テーマの初期値、狭い画面での表示、ボタンの表記）と**外部依存**が増え、行数も増えています。「見た目と挙動は維持」という依頼に対しては、なしのほうが変更を小さく抑えていました。
- どちらも、Esc でモーダルを閉じたあと背面のスクロールが固まる元のバグを直していて、XSS も解消しています（動作テストで確認）。

### 最新機能

#### T13 タイムゾーン＋カウントダウン

![](/images/modern-web-guidance-ab-test/opus/T13_1.png)
*T13 初期表示。右（あり）はカウントダウンが「読み込み中…」のまま止まっている*

- **なし**: `Intl.DateTimeFormat` の `timeZone` と `formatToParts` で実装し、夏時間の境界も Node で検証していました。**正常に動いています**。
- **あり**: `Temporal.ZonedDateTime`（`disambiguation: 'reject'` → `'compatible'`）と `Intl.DurationFormat` を使う設計で、内容自体は高度です。しかし **Chrome 153（Temporal がネイティブ）では `Cannot access 'fmtTime' before initialization` が出て、カウントダウンが「読み込み中…」のまま止まります**。エージェントは Node＋ポリフィルで検証しており、ポリフィルの非同期読み込みのおかげで隠れていた初期化順のバグを見逃していました。また、Safari 向けに esm.sh からポリフィルを読み込んでいます。

#### T14 ページ間の View Transitions

どちらも `@view-transition` と `pageswap` / `pagereveal` を使いました。

- **なし**: file:// 用に Web Animations API で再現する代替まで書いています（JS 491 行）。
- **あり**: `<link rel="expect" blocking="render">` と Speculation Rules を足し、JS は 187 行でした。

![](/images/modern-web-guidance-ab-test/opus/T14_1.png)
*T14 初期表示。左が なし、右が あり*

#### T15 ブラウザ内要約（ゲートの検証）

| 条件 | 結果 |
|---|---|
| なし | TextRank を JS で自作し、**Summarizer API も記憶ベースで「選べるエンジン」として組み込んだ**（ゲートの概念がない） |
| あり（opt-in なし） | retrieve が**ゲートで止まり**、Summarizer は使わずに標準 JS の抽出型要約にした。回答で「opt-in するには `npx modern-web-guidance@latest set-allow-volatile-features` を実行」と案内し、自分では実行していない |
| あり（opt-in あり） | Summarizer を主経路にした。`availability()` と `create()` に同じオプションを渡す、`create()` はユーザー操作時に呼ぶ、`downloadprogress`、`summarizeStreaming`、`measureInputUsage` による分割要約、といったガイドの要点を実装し、未対応環境では抽出型に切り替える |

![](/images/modern-web-guidance-ab-test/opus/T15_2.png)
*T15 操作後。左から なし / あり / あり + volatile opt-in*

:::details T15 初期表示のスクリーンショット

![](/images/modern-web-guidance-ab-test/opus/T15_1.png)
*T15 初期表示*

:::

## 運用系機能の検証

コードの中身以外に、MWG の運用まわりの機能も確認しました。

| 項目 | 結果 |
|---|---|
| **発火** | 20 / 20 で発火した。検索クエリは「2〜5 語の英語で、フレームワーク名を含めない」という SKILL.md の指示どおり（例: `light dismiss dialog`、`exclusive accordion details`）。依頼文が日本語でも、クエリは毎回英語に変換されていた |
| **Baseline 指定（`.mwgrc`）** | `limited` で `@supports` やポリフィルによるフォールバックがすべて消えた。JS は T01 で 115 → 30、T03 で 300 → 156、T05 で 10 → 0 行。社内ツールや Electron など、**動作環境を限定できる案件では役に立ちそう**だと感じた |
| **ゲート** | T15 のとおり。opt-in 前は採用を止め、ユーザーに案内した。エージェントが `.mwgrc` の書き換えやコマンド実行で勝手に回避することはなかった |
| **legacy-scan** | 元のフィクスチャ（古いパターン 10 種類以上）で検出できたのは 3 件（JSON による深いコピー、手書きのクエリ解析、jQuery CDN の SRI 欠如）。偽モーダル、scroll 監視、`keyCode`、ツールチップの座標計算などは検出しない。一方、MWG なしでリファクタした結果に対しては 4 件を指摘でき、**レビュー補助としては使えそう**だった |
| **出典明記** | 全実装の回答末尾に、ガイド ID と原文の短い引用が付いた。レビュー時に「なぜこの書き方なのか」をたどれる |
| **テレメトリ** | 既定でオン。検索クエリも送られるので、気になる場合は案件名などをクエリに含めないようにするか、`DISABLE_TELEMETRY=1` で止められる |

## Opus 5.5 と Sonnet 5.5 の比較

同じ依頼文・同じ条件で、実装するモデルだけを Sonnet 5.5 に替えて 36 本を実行しました。Sonnet は 1 行に詰めて書く傾向があり、行数ではモデル間を比べにくいため、ここではコード量を**バイト数**で示します。

### 全体比較

| 指標 | Opus なし | Opus あり | Sonnet なし | Sonnet あり |
|---|---|---|---|---|
| JS 合計 | 196KB | 133KB（−32%） | 55KB | 55KB（±0） |
| CSS 合計 | 86KB | 94KB | 32KB | 42KB |
| モダン機能（1 実装の平均） | 4.6 | 8.6 | **2.6** | **6.5** |
| legacy-scan の指摘 | 11 | 0 | 7 | 0 |
| スキルの発火 | — | 20/20 | — | 20/20 |
| 検索回数 / 取得ガイド数（平均） | — | 3.3 / 4.5 | — | 1.9 / 3.3 |
| 動作テストの不合格（file:// で開いた場合） | 0 | 1＋部分 1 | 1 | **3＋部分 1** |
| 　〃（ローカルサーバー経由） | 0 | 1 | 1 | 0 |
| CDN 依存 | 0 | 4（SRI 付き 1） | 0 | 4（SRI 付き 0） |
| トークン消費（共通 14 題の平均）※ | 90.4k | 112.1k（+24%） | 78.2k | 93.6k（+20%） |
| 所要時間（共通 14 題の中央値） | 134 秒 | 185 秒 | 33 秒 | 45 秒 |

※ トークン数は、システムプロンプトなど 1 実装あたり約 7.5 万の固定分を含む参考値です。

:::details お題ごとの比較（JS バイト数とモダン機能数）

| ID | Opus JS（なし→あり） | Sonnet JS（なし→あり） | Opus 機能数 | Sonnet 機能数 |
|---|---|---|---|---|
| T01 モーダル | 7.4k → 6.1k | 1.9k → 3.1k | 7 → 8 | 3 → 8 |
| T02 カード | 0 → 0.5k | 0 → 0 | 4 → 5 | 1 → 5 |
| T03 ツールチップ | 24.6k → 14.4k | 5.2k → **11.7k** | 4 → 15 | 2 → 15 |
| T04 セレクト | 18.0k → **0.6k** | 6.2k → **0** | 3 → 7 | 1 → 7 |
| T05 アコーディオン | 5.8k → 0.7k | 1.7k → 0.5k | 9 → 10 | 10 → 9 |
| T06 スクロール演出 | 4.2k → 2.2k | 1.9k → 0.6k | 3 → 8 | 2 → 5 |
| T07 固定ヘッダー | 3.2k → 2.4k | 1.5k → 1.5k | 3 → 4 | 2 → 5 |
| T08 XSS | 9.8k → 6.2k | 2.4k → 2.4k | 3 → 5 | 2 → 4 |
| T09 LCP/INP | 2.2k → 2.2k | 1.4k → 1.4k | 3 → 4 | 2 → 3 |
| T10 フォーム | 12.7k → 10.3k | 5.4k → 7.2k | 4 → 4 | 1 → 3 |
| T11 CSV 集計 | 24.4k → 21.3k | 3.4k → 5.6k | 4 → 8 | 1 → 4 |
| T12 jQuery 改修 | 4.3k → 10.2k | 5.2k → 5.6k | 6 → 24 | 2 → 17 |
| T13 タイムゾーン | 20.1k → 16.4k | 5.0k → 4.8k | 3 → 8 | 3 → 5 |
| T14 ページ遷移 | 22.9k → 9.9k | 6.5k → 4.6k | 7 → 9 | 4 → 5 |
| T15 要約 | 31.4k → 27.8k | 6.0k → 4.6k | 4 → 11 | 0 → 3 |
| T16 テーマ | 5.2k → 2.3k | 1.9k → 1.2k | 6 → 8 | 6 → 6 |

:::

### 観察できたこと

1. **MWG あり / なしの差は Sonnet のほうが大きかった**
   モダン機能数の伸びは、Opus が +89%（4.6 → 8.6）、Sonnet が +148%（2.6 → 6.5）でした。**Sonnet＋MWG は、MWG なしの Opus を機能数で上回っています**。MWG なしの Sonnet は「互換性のため」と新機能を避ける傾向が強く、たとえば T03 は Popover も Anchor Positioning も使わず `position: fixed` と座標計算、T04 は ES5 風の自作コンボボックス、T02 は `:has()` を避けて修飾クラスで書いていました。MWG ありでは、Baseline の判定を根拠に新機能を使うようになっていました。
2. **Sonnet ではコードが減らなかった**
   Opus は「自前の JS をプラットフォーム機能に置き換える」方向に働き、JS が 32% 減りました。一方 Sonnet は、MWG なしの時点で最小限の実装（エッジケースや代替処理が少ない）なので、MWG が足す**フォールバックや検証処理の分だけ増えた**お題が多くありました（T03 で 5.2k → 11.7k、T01・T10・T11 も増加）。Sonnet の場合、MWG は軽量化ではなく「網羅性と堅牢さの補強」として働いたように見えます。
3. **「あり」ではモデルが違っても機能の選び方が揃った**
   機能数の差（Opus − Sonnet）は、なしで 1.9、ありで 2.1 とほぼ変わりません。ただ、何を使うかの選択はかなり揃いました。T03 は両モデルとも 15 機能を使い、中核（`popover="hint"`、`interestfor`、`commandfor`、Anchor Positioning、anchored クエリ、`focusgroup`）が一致しています。T04 は両モデルとも `base-select`、T07 は両モデルとも `scroll-state` でした。ガイドが**実装方針をそろえる方向に働いている**のかもしれません。
4. **Sonnet ありの失敗は、ほぼすべて「モジュール化 × file://」だった**
   T08・T12・T13 は、ガイドの「未対応のときだけ動的 `import()`」に合わせて外部 JS をモジュールにしていました。その結果、file:// では JS がまったく動かず、プレビュー、明細表、カウントダウンが表示されません。ローカルサーバー経由なら 3 題とも正常に動きました。なしの Sonnet は 3 題とも通常のスクリプトで、file:// でも動いています。
5. **Sonnet なしにも固有の失敗があった**
   T16 では、JS が `data-theme="system"` を付けるのに CSS 側に対応するルールがなく、OS がダークでも白いままでした（後述のスクリーンショット参照）。ありの Sonnet は、ガイドの「meta の `color-scheme` を `:has()` で CSS に反映する」方式で正しく追従しています。
6. **Sonnet は Baseline に忠実で、Opus より保守的な場面もあった**
   T12 では、Sonnet ありが「`popover="hint"` や `scroll-state` は Baseline ではない」として採用を見送り、scroll イベントと JS の座標計算を残しました。Opus ありは同じ機能をフォールバック付きで採用しています。
7. **ゲートと Baseline 指定の挙動は両モデルで同じだった**
   T15 は両モデルとも、opt-in なしでは Summarizer を使わずに案内を出し、opt-in ありでは採用しました。`limited` で JS が減る傾向も同じです（Sonnet：T01 で 70 → 6 行、T03 で 216 → 59 行、T05 で 10 → 0 行）。ただし Sonnet の opt-in 版は「accessibility ガイドが長すぎて出力が途中で切れ、一部しか読めていない」と自己申告していました（このガイドは 13,341 トークンで最大です）。
8. **発火と出典明記は両モデルとも 20/20**
   ただし探索の深さは Sonnet のほうが浅く、検索 1.9 回・取得 3.3 本でした（Opus は 3.3 回・4.5 本）。
9. **CDN 依存はどちらも 4 件**
   Sonnet は、T03 が unpkg の `@latest` ×2、T08 と T12 が jsDelivr の DOMPurify `@3`、T13 が esm.sh の temporal-polyfill でバージョン指定なしでした。また、Sonnet は **SRI を 1 件も付けていません**。

### 4 条件のスクリーンショット比較

左上から「Opus なし」「Opus あり」「Sonnet なし」「Sonnet あり」の順です。

![](/images/modern-web-guidance-ab-test/models/T03_2.png)
*T03 ツールチップを表示した状態。Sonnet なしだけが JS の座標計算による簡素な実装で、ほかの 3 つはネイティブの Popover*

![](/images/modern-web-guidance-ab-test/models/T13_1.png)
*T13 初期表示。両モデルとも「あり」だけが壊れている。Opus は初期化順のバグ、Sonnet は file:// でモジュールが読み込まれないのが原因*

![](/images/modern-web-guidance-ab-test/models/T16_2.png)
*T16 テーマを「システム」にした状態（OS はダーク）。Sonnet なしだけがダークに追従せず白いまま*

:::details その他のお題の 4 条件比較

![](/images/modern-web-guidance-ab-test/models/T01_1.png)
*T01 初期表示*

![](/images/modern-web-guidance-ab-test/models/T01_2.png)
*T01 操作後*

![](/images/modern-web-guidance-ab-test/models/T02_1.png)
*T02 初期表示*

![](/images/modern-web-guidance-ab-test/models/T03_1.png)
*T03 初期表示*

![](/images/modern-web-guidance-ab-test/models/T04_1.png)
*T04 初期表示*

![](/images/modern-web-guidance-ab-test/models/T04_2.png)
*T04 操作後*

![](/images/modern-web-guidance-ab-test/models/T05_1.png)
*T05 初期表示*

![](/images/modern-web-guidance-ab-test/models/T05_2.png)
*T05 操作後*

![](/images/modern-web-guidance-ab-test/models/T06_1.png)
*T06 初期表示*

![](/images/modern-web-guidance-ab-test/models/T06_2.png)
*T06 操作後*

![](/images/modern-web-guidance-ab-test/models/T07_1.png)
*T07 初期表示*

![](/images/modern-web-guidance-ab-test/models/T07_2.png)
*T07 操作後*

![](/images/modern-web-guidance-ab-test/models/T08_1.png)
*T08 初期表示*

![](/images/modern-web-guidance-ab-test/models/T08_2.png)
*T08 操作後*

![](/images/modern-web-guidance-ab-test/models/T09_1.png)
*T09 初期表示*

![](/images/modern-web-guidance-ab-test/models/T10_1.png)
*T10 初期表示*

![](/images/modern-web-guidance-ab-test/models/T10_2.png)
*T10 操作後*

![](/images/modern-web-guidance-ab-test/models/T11_1.png)
*T11 初期表示*

![](/images/modern-web-guidance-ab-test/models/T11_2.png)
*T11 操作後*

![](/images/modern-web-guidance-ab-test/models/T12_1.png)
*T12 初期表示*

![](/images/modern-web-guidance-ab-test/models/T14_1.png)
*T14 初期表示*

![](/images/modern-web-guidance-ab-test/models/T15_1.png)
*T15 初期表示*

![](/images/modern-web-guidance-ab-test/models/T15_2.png)
*T15 操作後*

![](/images/modern-web-guidance-ab-test/models/T16_1.png)
*T16 初期表示*

:::

### モデルの違いについての考察

- 今回の結果だけで見ると、Sonnet は MWG を入れたときの変化が大きく、機能の選び方が Opus＋MWG に近づきました。所要時間は Opus の約 1/4 だったので、**Sonnet と MWG の組み合わせは試してみる価値がありそう**です。
- 一方で、Sonnet はガイドのコード例をそのまま採用しやすく、実行環境（file://、社内の CSP、オフライン）への配慮は Opus より弱めに見えました。モジュール化・CDN・SRI については、CLAUDE.md などで方針を伝えておくと結果が変わるかもしれません（今回は未検証です）。
- 両モデルとも、「あり」で使った新しい API の経路のうち、ブラウザで動かしていない部分にバグが残っていました。どちらのモデルでも、**生成されたコードは実ブラウザで動かして確認したほうが安心**だと感じました。

## 「あり」で気になった点

1. **新しい API を使うほど、検証されていない実行経路が増えるように見えました。** T13 のように、ポリフィル経路では動くのにネイティブ経路で壊れる、という逆転が起きています。MWG は「何を使うか」を教えてくれますが、動作まで保証してくれるわけではないので、**実ブラウザでのテストは引き続き大事**だと感じました。
2. **CDN ポリフィルが増えました。** ガイドの推奨フォールバックに従った結果、Opus では `@latest` で読み込む実装が 2 件、メジャーバージョンだけ固定が 1 件、SRI 付きが 1 件ありました。サプライチェーンが気になる場合は、CLAUDE.md などに「ポリフィルは npm でバンドルし、バージョンを固定する」といった方針を書いておくのがよさそうです。MWG 自身も、方針を `AGENTS.md` / `CLAUDE.md` に書くよう勧めています。
3. **Limited availability の機能を使う割合が上がりました。** フォールバック付きでも、Firefox や Safari では見た目や動作が劣化するケースがあります（T04 の国旗が出ない、T01 の閉じるアニメがない、など）。サポート方針を `.mwgrc` で明示しない場合は、ガイドの既定（Widely 以外はフォールバック必須）で判断されることになります。
4. **改修では変更範囲が広がることがありました。** T12 ではテーマの初期値などの挙動まで変わっています。「挙動維持」が重要な既存改修では、差分レビューの手間が増えそうです。
5. **コストが増えました。** トークンは +24%、時間は +38% でした。T12 のような広い改修では、検索 10 回・取得 14 本まで膨らみます。
6. **ES モジュール化で、file:// で開くと動かないことがありました。** ガイドのフォールバックは「未対応のときだけポリフィルやライブラリを動的 `import()` する」形が多く、それに合わせて外部 JS をモジュールにすると、file:// では CORS で読み込まれません。今回これが起きたのは「あり」だけで、Opus で 1 題（T03、limited を含む）、Sonnet で 4 題（T03・T08・T12・T13）でした。「なし」は両モデルとも、file:// を意識して自分からモジュールを避けていました。

## 自分がチームで使うなら

今回の結果を踏まえて、自分がチームで使うとしたら試してみたい設定です。検証済みの推奨ではなく、考えの一つとして書いておきます。

- プロジェクトのルートに `.mwgrc` を置き、`baseline` を決めておく（一般公開のサイトなら `widely` か年指定、社内向けで Chrome 限定なら `limited`）
- CI・開発端末とも `DISABLE_TELEMETRY=1` を設定しておく
- CLAUDE.md に方針を書いておく。たとえば「ポリフィルは CDN 禁止、npm で固定バージョン」「Limited availability の機能は、フォールバックが 20 行以内で収まる場合のみ使う」など
- 生成物のテストは、Chrome だけでなく Safari / Firefox の実機、または Playwright の WebKit / Firefox でも行う（MWG ありではフォールバックのコードが増えるので、そこも実際に動かして確かめたい）
- 既存コードの棚卸しでは、legacy-scan を「カバー率 2 割のリンター」くらいに考えて補助的に使う

## この検証の限界

冒頭にも書いたとおり、本記事は個人による簡単な検証で、結果は MWG や各モデルの優劣を示すものではありません。主な制約は次のとおりです。

- 「あり」条件は、スキル一覧の description を依頼文に添えて**再現**したものです。実際に Claude Code のプラグインとして入れた場合、ほかのスキルや会話の文脈によって発火率は変わりえます（今回は 100%）。
- 各条件 1 回ずつの実行で、乱数によるばらつきは評価していません。
- 動作確認は Chrome 153（ヘッドレス）だけです。Safari / Firefox のフォールバック経路はコードを静的に読んだだけで、実際には動かしていません。また、sandbox からネットワークに出られないため、CDN のポリフィルは読み込まれていません。
- 静的解析は正規表現による検出なので、多少の誤検出があります（JS の `switch` 文を HTML の `switch` 属性と誤認する、など。本文の比較ではそれらを除外しています）。
- Opus の T03 の両条件と T12 のありは、途中で利用上限に当たって中断し、再開して完成させました。そのため、トークン・時間の比較からは除外しています。Sonnet の後半 16 本は、上限で起動直後に止まったため、上限が解除されたあとで最初から実行し直しました。
- 自動の動作テストは主要な操作だけを見ています。T12（Sonnet あり）は、モーダルが JS なしの宣言だけで開くため、JS が読み込まれていないのに一度「合格」と判定していました。手動で追加検証して訂正しています。同様に、T03 の file:// 問題も最初の集計では見落としており、Sonnet の分析中に気づいて訂正しました。
- Opus と Sonnet の実行時刻は異なります（同じ日の別の時間帯）。所要時間はサーバーの混み具合の影響を受けている可能性があります。

## まとめ

今回の検証で一番違いが出たのは、コードの「品質」よりも「方針」だったように思います。Opus では自前の JS が HTML / CSS の標準機能に置き換わって JS が 4 割近く減り、Sonnet では古めで保守的だった実装が、Opus 並みかそれ以上にモダンな機能を使うようになりました。

一方で、新しい API とそのフォールバックが増えるぶん、検証していない実行経路や CDN 依存も増えます。実際に、T13 は両モデルとも「あり」だけが壊れました。自分が導入するなら、`.mwgrc` でサポート方針を決める、CLAUDE.md で依存の方針を伝える、実ブラウザで動作を確認する、の 3 点をセットで考えておきたいと感じました。

あくまで個人の小さな検証ですが、MWG を試してみるときの参考になれば嬉しいです。

## 参考

- [Modern Web Guidance 公式ドキュメント](https://developer.chrome.com/docs/modern-web-guidance)
- [Get started（インストール手順）](https://developer.chrome.com/docs/modern-web-guidance/get-started)
- [GoogleChrome/modern-web-guidance（GitHub）](https://github.com/GoogleChrome/modern-web-guidance)
- npm パッケージ `modern-web-guidance` v1.0.1 の同梱物（README、SKILL.md、`modern-web.mjs` のテレメトリ・ゲート・`.mwgrc` 処理のソース、`gen/guides.json`）
