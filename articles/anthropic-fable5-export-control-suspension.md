---
title: "米政府の輸出規制でClaude Fable 5が全世界停止――AI業界に衝撃が走った13日間"
emoji: "🚨"
type: "tech"
topics: ["AI", "Anthropic", "Claude", "輸出規制", "生成AI"]
published: false
---

## 何が起きたのか

2026年6月12日午後5時21分（米東部時間）、Anthropicは米国政府から輸出規制指令を受領した。指令の内容は、**Claude Fable 5およびClaude Mythos 5へのアクセスを、米国内外を問わず全外国人（Anthropicの外国人従業員を含む）に対して即座に停止せよ**というものだった。

Anthropicはこの指令に従い、世界中の全ユーザーに対して両モデルへのアクセスを遮断した。数十のグローバルクラウドプラットフォーム（AWS Bedrock、Google Cloud、Microsoft Foundryなど）でリアルタイムに国籍によるフィルタリングを行うことが技術的に不可能だったため、外国人だけでなく全顧客を対象とした一律停止となった。Fable 5はわずか3日前の6月9日にリリースされたばかりの最新フラグシップモデルであり、Mythos 5は約50の信頼された組織に限定提供されていたさらに上位のモデルだ。

6月25日現在、**両モデルは依然としてオフラインのまま**であり、復旧日は発表されていない。

## 政府の主張とAnthropicの反論

米政府は「Fable 5のジェイルブレイク（安全機構の回避）手法が発見された」ことを根拠に、国家安全保障上の措置としてこの指令を発出した。

しかしAnthropicの見解は大きく異なる。Anthropicによれば、政府から示されたのは**「特定のコードベースを読んでソフトウェアの欠陥を修正させる」という限定的なジェイルブレイク手法の口頭での説明**のみだった。Anthropicはこれを「既知の軽微な脆弱性に対する、狭い範囲の非汎用的なジェイルブレイク」と評価している。

Anthropicは公式声明で次のように警告した。

> この基準が業界全体に適用されれば、すべてのフロンティアモデルプロバイダーによる新モデルの展開が事実上停止することになる

## なぜ重要なのか

この事件が持つ意味は、単一企業の問題をはるかに超えている。

**1. 前例のない政府介入**
商用展開済みのAIモデルを政府命令で強制停止させた事例は初めてだ。「数億人が利用するサービスを一晩で止められる」という事実は、AI業界全体のリスク認識を根本から変える。

**2. 輸出規制の新たな適用範囲**
従来の輸出規制は半導体やハードウェアが中心だった。今回の指令はソフトウェア（AIモデル）に対して直接適用された点で、規制の射程が大幅に拡大したことを示している。

**3. 開発者・企業への実務的影響**
Fable 5に依存していたアプリケーションやワークフローは突然動かなくなった。6月22日には本来予定されていた課金体系の移行期限が過ぎたが、停止中のためAnthropicからの案内は出ていない。代替としてClaude Opus 4.8やSonnet 4.6への切り替えが推奨されている。

## 同時期のAI業界の動き

Fable 5の停止が注目を集める一方で、AI業界では他にも大きな動きがあった。

- **DeepSeek V4-Pro**：1.6兆パラメータのMoEモデルを、NVIDIAチップを一切使わずHuawei Ascend 950で訓練。米国の輸出規制下でも中国が独自にフロンティアモデルを構築できることを実証した
- **xAI Grok 4.3**：Amazon Bedrockで一般提供開始。100万トークンのコンテキストウィンドウを搭載
- **Apple iOS 27 Extensions**：WWDC 2026で発表されたフレームワークにより、SiriのバックエンドにClaude、ChatGPT、Geminiなどサードパーティ AIを設定可能に
- **EU Cloud and AI Development Act（CADA）**：欧州委員会が6月3日にクラウド主権フレームワークを提案

## 今後の注目点

Fable 5の復旧時期は依然不透明だ。TechTimesは6月21日に「Android アプリでFable 5が一時的に再出現した」「NSA関連の議会証言がモデル停止の方向性を変える可能性がある」と報じたが、これらは他のメディアでは確認できていない（※未検証情報）。

開発者が注視すべきポイントは以下の通りだ。

- 米商務省からの正式な復旧判断の発表
- 他のAIプロバイダー（OpenAI、Googleなど）に同様の指令が拡大するかどうか
- Anthropicが法的に異議を申し立てる可能性

AIモデルが「いつでも止められるインフラ」になり得るという現実を、今回の事件は突きつけている。マルチプロバイダー戦略やフォールバック設計の重要性が改めて浮き彫りになった。

---

### 主要ソース

- [Anthropic公式声明](https://www.anthropic.com/news/fable-mythos-access)
- [Fortune: Anthropic disables Fable and Mythos AI models](https://fortune.com/2026/06/13/anthropic-disables-fable-mythos-export-controls-national-security-threat/)
- [Al Jazeera: US asks Anthropic to block global access to top AI models](https://www.aljazeera.com/news/2026/6/14/us-asks-anthropic-to-block-global-access-to-top-ai-models-why-it-matters)
- [Time: Anthropic Pulls Its Most Powerful AI Models](https://time.com/article/2026/06/13/anthropic-fable-mythos-ban-US-security/)
- [The Hacker News: US Orders Anthropic to Suspend Fable 5 and Mythos 5](https://thehackernews.com/2026/06/us-orders-anthropic-to-suspend-fable-5.html)
- [National Law Review: Anthropic Suspends Access Following US Export Control Directive](https://natlawreview.com/article/ai-company-anthropic-suspends-access-claude-fable-5-claude-mythos-5-following-us)
- [TechTimes: Fable 5 Ban Update](https://www.techtimes.com/articles/318760/20260620/fable-5-ban-update-trump-softens-directive-stands-refund-deadline-closes-today.htm)（※一部未検証情報あり）
