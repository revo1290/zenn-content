---
title: "OpenAIが初の自社チップ「Jalapeño」を発表 ― Broadcomと共同開発、推論コスト50%削減へ"
emoji: "🌶️"
type: "tech"
topics: ["AI", "OpenAI", "半導体", "推論", "LLM"]
published: false
---

## 何が起きたか

2026年6月24日、OpenAIとBroadcomは共同で**「Jalapeño（ハラペーニョ）」**を発表した。OpenAI初のカスタムシリコンチップであり、LLM推論に特化した専用プロセッサだ。

設計はOpenAI、製造はBroadcomが担当。初期スケッチからファブリケーション準備完了まで約9ヶ月という異例のスピードで開発された。注目すべきは、チップ設計プロセスの一部にOpenAI自身のLLMが活用されたという点だ。

Broadcom CEOのHock Tan氏によれば、従来のAI用GPUと比較して**推論コストを約50%削減**できるとの初期テスト結果が出ている。2026年末までに本番データセンターへの配備を目指す。

発表を受け、Broadcom株は約2%上昇。一方でNvidiaは小幅に下落した。

## なぜこれが重要か

### Nvidiaの独占構造への直接的な挑戦

現在、AI推論・学習のほとんどがNvidiaのGPUに依存している。OpenAIのようなフロントランナーが自社チップに動くことは、この独占構造に風穴を開ける可能性がある。Googleの「TPU」、Amazonの「Trainium/Inferentia」に続く流れだが、OpenAIが参入する意味は大きい。世界最大のLLMプロバイダーが「自分たちの推論負荷を最も理解している」からだ。

### 推論コスト削減はユーザーにも波及する

ChatGPTやAPIの利用コストは、大半が推論インフラのコストに依存している。50%のコスト削減が実現すれば、API料金の引き下げやフリーティアの拡充など、エンドユーザーへの還元が期待できる。特にGPT-5.5クラスの大規模モデルでは推論コストがボトルネックになっており、専用チップによる効率化のインパクトは極めて大きい。

### 「AIがAIチップを設計する」時代の到来

Jalapeñoの開発にOpenAIのLLMが使われたという事実は、**AIによる半導体設計の加速**を示唆している。チップ設計は従来2〜3年かかるプロジェクトだが、9ヶ月での開発はその常識を覆す。AI支援による設計反復の高速化は、今後の半導体業界全体に波及するだろう。

## 同時期の注目ニュース

### Google DeepMindから相次ぐ人材流出

6月18〜24日の6日間で、Google DeepMindから4名の著名研究者が退職した。

- **Noam Shazeer** ― Gemini共同リード、「Attention Is All You Need」共著者。OpenAIへ移籍
- **John Jumper** ― 2024年ノーベル化学賞受賞、AlphaFold開発者。Anthropicへ移籍
- **Jonas Adler / Alexander Pritzel** ― Geminiの主要貢献者。いずれもAnthropicへの移籍を計画

SignalFire VCの分析では、DeepMindのエンジニアがAnthropicに移る確率は逆方向の約11倍。AnthropicとOpenAIのIPOが近づく中、財務的インセンティブも大きな要因と見られる。

### ChatGPTの市場シェアが初めて50%を割る

Sensor Towerの「State of AI 2026」レポートによると、ChatGPTのグローバルAIアシスタント市場シェアは**46.4%**に低下。2022年11月のローンチ以降、初めて50%を下回った。Google Geminiが27.7%、Claude（Anthropic）が10.3%と追い上げている。月間アクティブユーザー数は依然11億人を超えるものの、Geminiのandroid OS統合やClaudeの急成長により、一極集中は終わりつつある。

## 未検証・要注意の情報

- Jalapeñoの50%コスト削減は「初期テスト結果」であり、本番環境での実測値ではない。量産後のパフォーマンスは未確定
- Claude Fable 5に対する米国政府の輸出管理措置（6月12〜18日）が報じられているが、その正確な範囲と法的根拠はソースにより微妙に異なる
- GoogleからAnthropicへの人材流出について、Google側の公式コメントは限定的

## 一次ソース

- [OpenAI公式発表](https://openai.com/index/openai-broadcom-jalapeno-inference-chip/)
- [Broadcom IR](https://investors.broadcom.com/news-releases/news-release-details/openai-and-broadcom-unveil-llm-optimized-intelligence-processor)
- [CNBC報道](https://www.cnbc.com/2026/06/24/openai-and-broadcom-reveal-jalapeno-first-ai-chip-in-partnership.html)
- [CNN報道](https://edition.cnn.com/2026/06/24/tech/openai-broadcom-jalapeno-ai-chip)
- [TechCrunch: Google DeepMind人材流出](https://techcrunch.com/2026/06/24/ai-researchers-continue-to-leave-google-for-its-rivals/)
- [Sensor Tower: ChatGPTシェア低下（Business Standard）](https://www.business-standard.com/technology/artificial-intelligence/chatgpt-market-share-slips-below-50-google-gemini-anthropic-claude-gain-ground-126061700765_1.html)
