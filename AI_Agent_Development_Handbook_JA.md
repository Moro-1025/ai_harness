---
title: "AIエージェント開発ハンドブック"
subtitle: "設計思想・判断基準・実装テンプレート"
version: "1.1"
updated: "2026-08-27"
language: "ja"
---

# AIエージェント開発ハンドブック

> AIエージェントを「賢いモデルを呼び出す機能」ではなく、  
> **失敗を検知し、安全に停止し、証拠を残しながら仕事を完了するソフトウェアシステム**として設計するための実務ハンドブック。

---

## シリーズ案内

本シリーズは、次の2冊で構成します。

- **本書: AIエージェント開発ハンドブック**  
  自作するAIエージェントシステムの設計、実装、評価、運用を扱います。共通原則の正本は本書です。
- **[AIコーディングエージェント活用ハーネス・ハンドブック](AI_Coding_Agent_Usage_Harness_Handbook_JA_v1.1.md)**  
  Codexなどの既成コーディングエージェントを、Repositoryと開発工程の中で安全・再現可能・検証可能に利用する方法を扱います。

AIエージェント自体を開発する場合は本書を中心に参照し、既成のコーディングエージェントを利用する場合は活用ハーネスを中心に参照してください。活用ハーネスは単体でも読めますが、共通原則の詳細は本書を正本とします。

---

## Version 1.1の主な変更

- NIST、OWASP、OpenTelemetry、Provider公式資料を追加し、主要主張と根拠の対応表を設けた
- 読者別の読書ルートと、リスク別の適用レベルを追加した
- 行動誘導、運用統制、Agentから独立した強制境界を区別した
- 証拠の強さを1本の段階ではなく、独立性・完全性・鮮度・改ざん耐性・再現性の複数軸で評価するよう変更した
- イベントログ、現在状態、業務データ、モデル向け要約、長期記憶の正本を分離した
- 並行実行、重複処理、外部処理の不確定結果、承認後の対象変更を扱う章を追加した
- セキュリティ脅威モデルを、SSRF、供給網、Memory/RAG汚染、Cross-tenant漏えい、Approvalすり替えまで拡張した
- 実行可能な最小参照実装とテストを追加した

---

## 読者別の読書ルート

| 目的 | 最初に読む範囲 |
|---|---|
| 最小の単一Agentを作る | 中核原則 → 第1章 → 第2章 → 第3章 → 第5章 → 第7章 → 第12章 |
| 本番運用へ進める | 上記 + 第6章 → 第8章 → 第9章 → 第10章 → 第13章 |
| RAG・Memoryを設計する | 第4章 → 第6章 → 第7章 → 第8章 → 第10章 |
| Multi-Agentを導入する | 第3章 → 第6章 → 第11章 → 第13章 |
| Security Reviewを行う | 第1章 → 第5章 → 第10章 → 第13章 → 付録C |
| 既存Agentを設計レビューする | 中核原則 → 付録A → 付録C → 付録F |
| 実装例から理解する | 第3章 → 第5章 → 付録B → 付録E |

---

## リスク別の適用レベル

すべてのAgentへ同じ統制を要求しません。次は説明用の分類であり、法的・業務上の正式なリスク区分ではありません。

| レベル | 代表例 | 最低限の統制 |
|---|---|---|
| 低リスク | 読取り、要約、候補作成、下書き | Scope、基本Validation、実行ログ、停止条件 |
| 中リスク | コード変更、可逆な設定変更、内部DBへの限定書込み | 隔離、差分、Test、Execution Budget、Idempotency、Rollback |
| 高リスク | 本番変更、外部送信、権限変更、個人情報、金銭、不可逆操作 | Agentから独立した強制境界、独立検証、Human Gate、監査、補償・緊急停止 |

リスクはModelの自信度では決めません。少なくとも次を見ます。

- 影響範囲(Blast Radius)
- 可逆性(Reversibility)
- 外部副作用
- 権限・Credential
- データ機密性
- 誤りの検知可能性
- 復旧時間
- 法的・契約上の責任

---

## 共通用語

2冊で共通して使用する用語は、次の意味へ統一します。完全版は付録Dを参照してください。

| 用語 | 本シリーズでの意味 |
|---|---|
| ハーネス(Harness) | モデルの周囲で、コンテキスト、ツール、権限、状態、検証、評価、可観測性、コスト、回復を管理する実行・運用層 |
| タスク契約(Task Contract) | 目的、期待状態、範囲、禁止事項、完了条件、必要証拠、停止条件を開始前に固定した契約 |
| 完了条件(Completion Criteria) | タスクを終了してよいと判定する、観測可能な条件 |
| 証拠(Evidence) | 完了条件を満たしたことを確認するための、コマンド結果、差分、ログ、テスト、観測記録など |
| 実行予算(Execution Budget) | Step、時間、Tool Call、費用、再試行、並列数などの実行上限 |
| 検証(Verification) | 特定の出力・変更・事後条件が期待どおりかを確認する処理 |
| 評価(Evaluation) | 代表タスク集合を使い、システム品質を比較・測定する処理 |
| レビュー(Review) | 設計、差分、証拠、残存リスクを別の視点で吟味する行為 |
| 状態(State) | 現在の進行段階、完了項目、未完了項目、承認、予算使用量など、再開に必要な情報 |
| コンテキスト(Context) | そのModel Callでモデルへ実際に提示する情報 |
| 記憶(Memory) | 将来のタスクで再利用するため、検証・期限・スコープを伴って保存した情報 |
| セッション(Session) | 会話、イベント、状態、成果物を継続的に関連付ける実行単位 |
| 引継ぎ(Handoff) | 別の人・Agent・Sessionへ、目的、状態、制約、証拠、次の操作を構造化して渡すこと |
| 人間ゲート(Human Gate) | 人間の判断または承認がなければ次へ進めない境界 |
| 停止条件(Stop Condition) | 完了、予算超過、同一失敗、権限不足、不確定結果など、実行を止める条件 |
| 完了報告(Completion Report) | 実施内容、完了条件、証拠、未実施事項、残存リスク、Rollbackをまとめた最終成果物 |

---

## 通し事例

本書では、必要に応じて次の事例へ設計原則を対応付けます。

> **期限切れRefresh Tokenを受け取ったAPIが、本来401を返すべきところ500を返す不具合を修正する。**

本書では、この事例をAIエージェントシステム内部の観点から扱います。

```text
Task Contract
→ 調査Tool
→ Agent Loop
→ 修正候補
→ File Write Tool
→ Targeted Test
→ Integration Test
→ Completion Criteria照合
→ Evidence付きCompletion Report
```

同じ事例を活用ハーネスでは、Repository、権限、Branch / Worktree、CI、独立Reviewの観点から扱います。

---

## 本書の目的

本書は、AIエージェント開発に関する複数の公開投稿で繰り返し扱われている設計思想と、実務で再利用できる判断基準・実装テンプレートだけを抽出し、体系化したものです。

対象は次のようなシステムです。

- 外部APIやデータベースを操作するAIエージェント
- ファイルを読取り・変更するコーディングエージェント
- 検索、調査、文書生成を複数ステップで行うエージェント
- 人間の承認を挟みながら業務を進めるエージェント
- 複数のサブエージェントを統合運用するシステム
- 長時間・複数セッションにまたがって状態を維持するシステム

本書には、個別製品の紹介、主要GitHubリポジトリのREADME要約、特定実装の内部構造解説は含めません。

---

## 参考情報の扱い

Version 1.1では、参考情報を次の順で扱います。

### 一次的な根拠

1. **NIST AI Risk Management Framework 1.0 / Playbook**  
   AIリスクをGovern・Map・Measure・Manageの継続的な活動として扱い、設計、評価、運用、第三者、Documentationをライフサイクル全体で管理するために参照します。AI RMFは任意利用のFrameworkであり、現在改訂作業中です。
2. **NIST AI 600-1: Generative AI Profile**  
   生成AI固有のリスクを、既存のAI RMFへ追加して扱うために参照します。
3. **OWASP AI Agent Security Cheat Sheet / OWASP Top 10 for Agentic Applications 2026 / OWASP GenAI LLM Top 10 2026**  
   Tool misuse、Prompt Injection、Memory poisoning、Identity / Privilege abuse、Supply chain、Unexpected code execution、Human-Agent trustなどの脅威と対策を整理するために参照します。
4. **OpenTelemetry Semantic Conventions**  
   Trace、Metric、Logの命名と相関を揃えるために参照します。GenAI固有のSemantic Conventionsは独立Repositoryへ移され、Development状態の項目を含むため、採用時にVersionを固定します。
5. **採用するModel・Cloud・Agent SDKの公式文書**  
   Tool schema、停止理由、Rate limit、Data handling、Authentication、Sandbox、Provider固有の制約は、一般論ではなく採用製品のCurrent docsを正本とします。

### 補助的な参考情報

公開投稿と[Agent Harness](https://agent-harness.ai/)の公開記事は、繰り返し現れる設計論点の抽出と、実務上の整理へ補助的に使用します。個別記事の性能値、改善率、閾値は普遍的な推奨値として採用しません。

本文の主要主張と参考文献の対応は、付録Fにまとめます。

---

## 表記方針

意味を保ったまま日本語化できる用語は、初出時に原則として**日本語(English)**で示します。

例：

- ハーネス品質(Harness Quality)
- 実行予算(Execution Budget)
- 完了条件(Completion Criteria)
- 可観測性(Observability)
- 冪等性(Idempotency)

次のものは、無理に日本語化しません。

- API名、設定キー、コマンド名、コード識別子
- LLM、RAG、MCP、TTFT、TPOT、SLOなど定着した略語
- 日本語化すると別概念と誤解されやすい専門用語

---

## 数値を含む例の読み方

本書の設定例やテンプレートに含まれる具体的な数値は、すべて**説明用の例**です。

> **重要:** `max_steps: 12`や`max_cost_usd: 1.00`などは、推奨値や安全保証値ではありません。実際の値は、タスク分布、リスク、モデル、ツール遅延、予算、SLO、実測結果に基づいて決めてください。

数値を含むコード例には、原則として次の注記を付けています。

```yaml
# 例: 初期検証用の仮値。実測後に調整する。
```

---

# 目次

1. [中核となる設計原則](#core-principles)
2. [第1章 システム境界と責任分離](#chapter-1)
3. [第2章 タスク契約・計画・完了条件](#chapter-2)
4. [第3章 エージェントループと停止制御](#chapter-3)
5. [第4章 コンテキスト設計](#chapter-4)
6. [第5章 ツール設計と実行統制](#chapter-5)
7. [第6章 状態・セッション・記憶](#chapter-6)
8. [第7章 検証・評価・レビュー](#chapter-7)
9. [第8章 可観測性・トレース・再生](#chapter-8)
10. [第9章 信頼性・コスト・性能・回復](#chapter-9)
11. [第10章 セキュリティと人間参加型制御](#chapter-10)
12. [第11章 マルチエージェント統合運用](#chapter-11)
13. [第12章 導入順序と本番移行](#chapter-12)
14. [第13章 並行実行・一貫性・不確定結果](#chapter-13)
15. [付録A 判断基準早見表](#appendix-a)
16. [付録B 実装テンプレート集](#appendix-b)
18. [付録D 共通用語集](#appendix-d)
19. [付録E 最小参照実装](#appendix-e)
20. [付録F 主要主張と参考根拠の対応](#appendix-f)
21. [付録G 参考文献](#appendix-g)

---

<a id="core-principles"></a>

# 中核となる設計原則

## 原則1　モデルとエージェントを同一視しない

LLMはエージェントを構成する一部です。

```text
エージェント
= モデル
+ コンテキスト構築
+ ツール
+ 状態
+ 実行ループ
+ 権限
+ 検証
+ 可観測性
+ 評価
```

モデルを交換しても、ツールや検証が壊れていればシステムは信頼できません。逆に、モデル能力が同じでも、ハーネス品質(Harness Quality)によって完了率、安全性、コスト、保守性は大きく変わります。

## 原則2　LLMは提案し、決定論的コードが許可する

LLMへ任せるのは、曖昧さを許容できる処理です。

- 意図理解
- 候補生成
- 分類
- 計画案
- 検索語生成
- 説明文生成

通常コード、ポリシーエンジン、権限管理層へ任せるものは次です。

- 型検証
- 認証・認可
- 金額上限
- データ整合性
- 一意性
- 冪等性
- 操作可能範囲
- 監査

基本原則は次です。

> **LLMは操作を提案できるが、実行権限を自分で付与してはならない。**

## 原則3　成功を期待するのではなく、失敗を制御する

AIエージェントは次のように失敗します。

- 誤ったツールを選ぶ
- 引数を作り間違える
- 同じエラーを繰り返す
- 完了していないのに終了する
- 終了すべきなのにループする
- 古い情報を使う
- 副作用を重複実行する
- 予算を使い切る

したがって、設計の中心は「どう成功させるか」だけではなく、次の問いになります。

```text
どこで失敗するか
→ どう検知するか
→ どう停止するか
→ どこまで戻すか
→ 誰へ引き継ぐか
→ 何を記録するか
```

## 原則4　作業ではなく成果を定義する

```text
作業:
- ファイルを書いた
- APIを呼んだ
- テストを追加した

成果:
- 要求された状態が成立した
- 既存挙動を壊していない
- 証拠で確認できる
```

「実装した」と「完了した」を分けます。

## 原則5　エージェントの自己申告より証拠を優先する

次は証拠ではありません。

- 「修正しました」
- 「正常に動くはずです」
- 「問題ありません」

証拠として扱えるものは次です。

- テスト結果
- API応答
- 差分
- スクリーンショット
- ログ
- 検証レポート
- 再現手順
- 監査イベント

## 原則6　コンテキストは有限資源として配分する

コンテキストウィンドウが大きくても、情報を無制限に入れると品質が上がるとは限りません。

- 指示競合が増える
- 重要事項が埋もれる
- 古い情報を参照しやすくなる
- コストと遅延が増える
- 検索結果のノイズが増える

コンテキストは「保存場所」ではなく、**今回の判断に必要な作業領域**です。

## 原則7　状態は会話の外にも保存する

長時間タスクでは、会話履歴だけに状態を置かないようにします。

- 計画
- 未完了項目
- 変更済み対象
- 検証結果
- 承認状態
- 再開位置

これらは、構造化状態、イベントログ、チェックポイントなどに保存します。

## 原則8　境界ごとに検証する

次の境界は、すべて失敗面(Failure Surface)です。

- モデル → ツール
- ツール → モデル
- エージェント → サブエージェント
- 検索 → 回答生成
- 計画 → 実行
- 実行 → 完了判定
- エージェント → 人間承認

後工程へ渡す前に、構造、必須項目、値域、出典、権限、鮮度を確認します。

## 原則9　可観測性を後付けしない

最終回答だけを保存しても、失敗原因は分かりません。

少なくとも次を追跡できるようにします。

- どのモデルを使ったか
- 何をコンテキストへ入れたか
- どのツールを、どの引数で呼んだか
- 何が返ったか
- 何回再試行したか
- どこで状態が変わったか
- 何に時間と費用を使ったか
- なぜ停止したか

## 原則10　評価対象はモデルではなくシステム全体

評価対象には、次が含まれます。

- モデル
- システムプロンプト
- コンテキスト構築
- 検索
- ツール説明
- ツール実装
- 再試行
- ルーティング
- 記憶
- 完了判定
- 人間承認

モデルだけを比較しても、本番のタスク成功率は分かりません。

## 原則11　不可逆な操作の直前に人間を置く

人間の承認は、すべてのステップに入れるのではなく、判断価値が高い場所へ置きます。

- 金銭
- 削除
- 外部送信
- 公開
- 権限変更
- 契約・法務
- 本番反映
- 個人情報利用

承認時には、差分、理由、代替案、影響、ロールバック方法を提示します。

## 原則12　複雑性は失敗が確認されてから追加する

最初から次をすべて導入しないようにします。

- マルチエージェント
- 長期記憶
- ベクトル検索
- 分散キュー
- 自己改善
- 複雑なワークフローエンジン

まず単一エージェントで失敗を観測し、原因に対応する機能だけを追加します。

## 原則13　改善されたモデルに合わせて制御を削除できるようにする

ハーネスは永続的に複雑化させるものではありません。

- 不要になった再試行を削除できる
- 不要になったプロンプト規則を削除できる
- モデル固有の回避策を切り離せる
- 検証層を交換できる
- ツールを無効化して副作用を残さない

**削除可能性(Design for Deletion)**を設計品質の一部として扱います。

## 原則14　制御の強制力を区別する

制御は、同じ「ルール」であっても強制力が異なります。

| 段階 | 目的 | 例 |
|---|---|---|
| 行動誘導 | Agentへ望ましい行動を伝える | Prompt、System Instruction、Skill、手順書 |
| 運用統制 | 通常フローで自動確認・中止する | Hook、Policy middleware、Validation Script、Approval UI |
| 独立した強制境界 | Agent自身が変更・無視できない境界で拒否する | OS、Container、Network Policy、Credential Scope、DB Authorization、CI、Protected Branch |

高リスク操作の安全性を、行動誘導だけへ依存させません。越えてはならない境界は、Agentの出力や作業Workspaceから独立した層で強制します。

---

<a id="chapter-1"></a>

# 第1章 システム境界と責任分離

## 1.1 4つの層を分ける

```text
モデル(Model)
  ↓
実行基盤(Runtime)
  ↓
ハーネス(Harness)
  ↓
エージェント製品(Agent Product)
```

| 層 | 主な責任 |
|---|---|
| モデル | 理解、推論、生成、ツール候補の提案 |
| 実行基盤 | プロセス、キュー、永続化、再開、同時実行 |
| ハーネス | コンテキスト、ツール、権限、検証、評価、可観測性、コスト制御 |
| エージェント製品 | 業務価値、UI、承認フロー、データ、監査、運用責任 |

## 1.2 責任分離表

| 判断・処理 | LLM | 通常コード | 人間 |
|---|:---:|:---:|:---:|
| 自然言語の意図理解 | 主 | 補助 | 必要時 |
| 操作候補の生成 | 主 | 検証 | 必要時 |
| 入力型の検証 | 不可 | 主 | 不要 |
| 権限判定 | 不可 | 主 | 例外承認 |
| 金額・件数上限 | 不可 | 主 | 上限変更 |
| 外部送信内容の作成 | 主 | 検査 | 高リスク時 |
| 本番反映 | 提案 | 実行制御 | 主 |
| 完了判定 | 候補 | 証拠確認 | 高リスク時 |
| 監査記録 | 不可 | 主 | 閲覧・監督 |

## 1.3 フルハーネスが必要になる条件

次のどれかに該当する場合、単純なチャットUIより強い制御が必要です。

- 外部APIを呼ぶ
- データを書き換える
- 複数ステップを自律実行する
- ユーザーが毎回出力を確認しない
- 長時間実行する
- 複数のモデルやツールを切り替える
- 個人情報や機密情報を扱う
- 金銭・権限・公開操作を行う
- 失敗時の再開が必要
- 結果の説明責任や監査が必要

## 1.4 制御の強制力を配置する

各要求について、必要な強制力を決めます。

```text
望ましい作法
→ Prompt / Skill

忘れた場合に自動検出したい
→ Hook / Validation / Policy middleware

絶対に越えてはならない
→ OS / Container / Network / Credential / Database / CI
```

例として「本番DBを削除しない」はPromptへ書くだけでは不十分です。本番Credentialを渡さない、対象DBへ接続できないNetworkに置く、DB Roleで`DROP`を拒否する、という独立境界へ移します。

# 第2章 タスク契約・計画・完了条件

## 2.1 タスク開始前に固定するもの

エージェントへ作業を始めさせる前に、最低限次を定義します。

1. 目的
2. 成果
3. 完了条件
4. 変更可能範囲
5. 禁止事項
6. 証拠
7. 判断が必要な事項
8. 停止条件

これを**タスク契約(Task Contract)**として扱います。

## 2.2 良い完了条件

良い完了条件は、第三者が確認できます。

悪い例：

```text
使いやすくする。
問題を直す。
品質を上げる。
```

良い例：

```text
期限切れトークンで401を返す。
有効なトークンの既存挙動を維持する。
回帰テストが成功する。
公開APIの形式を変更しない。
```

## 2.3 質問すべき曖昧さ

次の条件に該当する場合は、人間へ確認します。

- 選択によって公開仕様が変わる
- DBやデータ移行が変わる
- 後から戻すコストが高い
- ユーザーの価値判断が必要
- 法務、金銭、セキュリティに影響する
- 複数案が同程度に妥当

次の場合は、既定値を選び、仮定を明記して進められます。

- 容易に戻せる
- 局所的な実装詳細
- 既存規約から一意に決まる
- 完了条件へ影響しない

## 2.4 計画を成果物にする

複雑なタスクでは、計画を会話中の一時的な文章ではなく、レビュー可能な成果物として扱います。

計画に含めるもの：

- 対象範囲
- 変更単位
- 依存関係
- 検証方法
- ロールバック方法
- 未確定事項
- リスク

## 2.5 証拠設計(Evidence Design)

証拠の強さは、Testの種類だけでは決まりません。少なくとも次の軸で評価します。

| 軸 | 確認する問い |
|---|---|
| 関連性 | 完了条件そのものを検証しているか |
| 完全性 | 正常系だけでなく、対象となる失敗系・境界条件を含むか |
| 独立性 | Writer Agentの自己申告だけでなく、別Process・CI・人間が再確認したか |
| 改ざん耐性 | Agentが書込み可能な場所だけに証拠が置かれていないか |
| 鮮度 | 現在のCode、Config、Model、Tool Policyに対する結果か |
| 対象固定 | Commit、Artifact hash、入力、環境Versionへ結び付いているか |
| 再現性 | 同じ条件で再実行できるか |
| 不確実性 | 実行できなかった項目や観測限界を明記しているか |

### 証拠の生成元

| 種類 | 位置付け |
|---|---|
| Agent生成証拠 | 作業中の自己検証。速いが、Agent自身が選択・改変できる可能性がある |
| 決定論的検証証拠 | Test、Schema、Policy、Queryなど、同じ入力に対して機械的に確認した結果 |
| 独立実行証拠 | Clean Environment、Trusted CI、別権限Process、別Reviewerが再実行した結果 |
| 人間観測証拠 | UI、業務妥当性、高リスク判断、最終Acceptanceなど、機械検証だけでは不足する確認 |

すべてのタスクに最も強い証拠は必要ありません。リスク別の目安は次です。

| リスク | 例として求める証拠 |
|---|---|
| 低 | Agent生成証拠 + 基本Validation |
| 中 | 決定論的検証 + Diff / Artifact + 必要に応じたClean再実行 |
| 高 | Trusted CIまたは独立環境 + Human Gate + Commit / Action hashへ結び付いた監査証拠 |

## 2.6 完了判定

完了は次のすべてを満たした状態です。

```text
完了条件を満たす
AND
必要な証拠がある
AND
重大な未解決エラーがない
AND
変更範囲を逸脱していない
AND
残存リスクが報告されている
```

# 第3章 エージェントループと停止制御

## 3.1 推奨状態機械

```text
受付(Intake)
→ 重要事項確認(Clarify)
→ 計画(Plan)
→ 実行(Execute)
→ 検証(Verify)
→ 回復(Recover) または 人間へ引継ぎ(Escalate)
→ 完了(Complete)
```

各状態の責任を混ぜないことが重要です。

| 状態 | 主な責任 |
|---|---|
| intake | 要求・対象・制約を受け取る |
| clarify | 高影響の未確定事項を解消する |
| plan | 手順、依存、証拠、予算を決める |
| execute | ツールで状態を変更する |
| verify | 完了条件と事後条件を確認する |
| recover | 再試行、代替、ロールバックを選ぶ |
| escalate | 人間が判断できる情報をまとめる |
| complete | 証拠付きで終了する |

## 3.2 実行予算(Execution Budget)

ステップ数だけでなく、時間、費用、ツール回数、反復、進展を制限します。

```yaml
# 例: 小規模な検証環境向けの仮値。実運用値ではない。
execution_budget:
  max_steps: 12
  max_tool_calls: 24
  max_wall_time_seconds: 180
  max_input_tokens: 120000
  max_output_tokens: 16000
  max_cost_usd: 1.00
  same_action_limit: 2
  same_error_limit: 2
  no_progress_limit: 3
```

## 3.3 進展なし(No Progress)の判定

次のいずれかが続く場合、進展なしと判断します。

- 同じツールと同じ引数を繰り返す
- 同じファイル・ページだけを再読する
- 同じエラーコードが続く
- 状態ハッシュが変わらない
- 未完了項目が減らない
- 新しい証拠が増えない
- 同じ仮説を言い換えているだけ

進展判定は、モデルの自己評価だけに依存させません。

## 3.4 エラー分類

| 種別 | 例 | 基本対応 |
|---|---|---|
| 一時的(Transient) | 429、タイムアウト、一時的5xx | 制限付き再試行 |
| 入力不正(Invalid Input) | 必須項目不足、型不一致 | 引数を修正 |
| 権限不足(Permission) | 認可失敗 | 人間へ引継ぎまたは停止 |
| 業務規則違反(Policy) | 上限超過、対象外操作 | 停止または代替案 |
| 競合(Conflict) | バージョン競合、ロック | 再読込み後に再判断 |
| 永続的(Permanent) | 対象不存在、機能未対応 | 経路変更または停止 |
| 不明(Unknown) | 分類不能 | 安全側に停止・記録 |

## 3.5 再試行の原則

- 同じ引数を無条件で再送しない
- 再試行前にエラー分類を行う
- 副作用ツールでは冪等性を要求する
- 再試行回数を記録する
- 上限到達時は構造化エラーを返す
- 再試行で事態が悪化する場合は即停止する

```python
# 例: 説明用の擬似コード。回数・待機時間は実測後に調整する。
for attempt in range(3):
    result = call_tool(request)
    verdict = verify_tool_result(result)

    if verdict.passed:
        return result

    if not verdict.retryable:
        raise PermanentToolError(verdict.reason)

    sleep(min(2 ** attempt, 4))

raise RetryBudgetExceeded()
```

## 3.6 停止理由を機械判定可能にする

例：

```text
completed
budget_exceeded
no_progress
permission_required
verification_failed
user_cancelled
unsafe_operation
dependency_unavailable
unrecoverable_error
```

自由文だけで停止理由を残すと、集計・評価・自動回復が難しくなります。

## 3.7 最小ループの擬似コード

```python
state = load_or_create_state(task)

while True:
    enforce_budget(state)

    if completion_criteria_met(state):
        return complete_with_evidence(state)

    action = propose_next_action(state)
    validate_action(action, state.policy)

    result = execute_idempotently(action)
    record_event(action, result)

    verification = verify_result(action, result, state)
    update_state(state, action, result, verification)

    if verification.requires_human:
        return escalate_with_decision_packet(state)

    if state.no_progress_detected:
        return stop("no_progress", state)
```

# 第4章 コンテキスト設計

## 4.1 コンテキストの役割

コンテキストは、モデルがその時点で参照する作業領域です。

通常、次が含まれます。

- システム・安全指示
- 現在の目的
- 完了条件
- 現在の計画
- 最近の会話
- 関連コード・文書
- 検索結果
- 記憶
- ツール結果
- ツール定義

## 4.2 優先順位

推奨する一般的な優先順位は次です。

1. 安全・権限規則
2. 現在の目的と完了条件
3. 重要な制約
4. 現在の状態と未完了項目
5. 直接関係するコード・データ
6. 直近の検証結果
7. 関連する過去判断
8. 古い会話
9. 重複・雑談・低関連情報

## 4.3 コンテキスト予算(Context Budget)

```yaml
# 例: 128k級のコンテキストを想定した説明用配分。推奨値ではない。
context_budget:
  system_and_security_tokens: 6000
  task_and_constraints_tokens: 6000
  active_state_tokens: 5000
  relevant_sources_tokens: 42000
  recent_history_tokens: 18000
  tool_results_tokens: 18000
  answer_reserve_tokens: 20000
  emergency_reserve_tokens: 13000
```

数値より重要なのは、次です。

- 何を優先して残すか
- 何を外部へ退避するか
- いつ圧縮するか
- 圧縮後も何を保持するか

## 4.4 常時コンテキストと必要時コンテキスト

### 常時含める

- 安全規則
- 不変の設計制約
- 必須の完了定義
- 重要な禁止事項
- 現在の目的

### 必要時だけ読む

- 長い操作手順
- 特定領域のスタイルガイド
- リリース手順
- インシデント対応手順
- 特定機能の詳細設計
- 過去セッション全文

「忘れないように全部入れる」は、コンテキスト汚染(Context Pollution)を生みます。

## 4.5 コンテキスト圧縮(Compaction)

圧縮対象：

- 完了済みの探索ログ
- 大きなツール結果
- 重複したコード断片
- 解決済みエラー
- 古い計画
- 参照済みの全文データ

圧縮後も残すもの：

- 原要求
- 完了条件
- 未完了項目
- 採用判断と理由
- 却下案と理由
- 変更対象
- 検証結果
- 現在の障害
- 出典参照ID

## 4.6 ツール結果の外部退避

```text
ツール結果
├─ モデルへ渡す: 要約、重要箇所、出典ID、次の選択肢
└─ 外部保存: 全文、ログ、ファイル、バイナリ、巨大JSON
```

モデルが必要な場合だけ、参照IDから再読込みします。

## 4.7 出典と鮮度

コンテキストへ入る情報には、可能な限り次を付けます。

- source_id
- source_type
- retrieved_at
- updated_at
- authority
- confidence
- scope
- expires_at

これにより、古い記憶と新しい事実が競合したときに判断できます。

## 4.8 コンテキスト可視化

次を観測できるようにします。

- カテゴリ別トークン量
- 最大コンテキストに対する使用率
- ツール定義の占有量
- 検索結果の占有量
- 圧縮前後の差
- 不可視注入の内容
- キャッシュヒット率

# 第5章 ツール設計と実行統制

## 5.1 ツールはエージェントの操作面

モデルにとって、ツール名、説明、入力スキーマ、結果形式はUIです。

良いツールは次を満たします。

- 1ツール1責任
- 用途が名前から推測できる
- 使う条件と使わない条件が明確
- 入力範囲が狭い
- エラーが構造化される
- 結果が大きすぎない
- 副作用の有無が明確

## 5.2 ツール実行パイプライン

```text
ツール候補
→ 入力スキーマ検証
→ 認証
→ 認可
→ 業務規則
→ リスク判定
→ 必要時の人間承認
→ 冪等性確認
→ 実行
→ 出力スキーマ検証
→ 事後条件確認
→ 監査記録
```

## 5.3 入力検証

入力検証には次を含めます。

- required
- type
- enum
- min / max
- length
- pattern
- allowed_resource
- tenant_scope
- path_scope
- destructive_flag

LLMが生成したJSONが構文上正しくても、業務上正しいとは限りません。

## 5.4 出力検証

HTTP 200でも失敗の場合があります。

- 本文が空
- 必須項目がない
- 対象IDが違う
- 更新件数が想定外
- 時刻が古い
- 結果が部分的
- エラーメッセージが本文に埋まっている

出力は、構造だけでなく**妥当性(Reasonableness)**と**事後条件(Post-condition)**を確認します。

## 5.5 副作用の分類

| リスク | 例 | 基本制御 |
|---|---|---|
| 読取り専用 | 検索、ファイル読取り | 入力検証、範囲制限 |
| 可逆 | 下書き作成、一時ファイル | 記録、Undo |
| 条件付き可逆 | DB更新、設定変更 | 事前スナップショット、補償処理 |
| 不可逆・外部影響 | 送信、公開、削除、送金 | 人間承認、冪等性、監査 |

## 5.6 冪等性(Idempotency)

再試行が発生し得るツールは、同じ操作を二重実行しない仕組みを持ちます。

```json
{
  "operation": "send_message",
  "idempotency_key": "task-123:step-7:recipient-456",
  "payload_hash": "sha256:..."
}
```

## 5.7 ツール結果の共通形式

```json
{
  "ok": true,
  "status": "completed",
  "data": {},
  "evidence_refs": [],
  "side_effects": [],
  "retryable": false,
  "error_code": null,
  "message": null
}
```

エラーも同じ形式で返し、モデルが機械的に扱えるようにします。

## 5.8 ツールの失敗方針

### 継続優先(Fail Open)が向くもの

- 表示補助
- 任意の文章整形
- 追加メタデータ
- 低リスクの補助機能

### 安全優先(Fail Closed)が向くもの

- 権限確認
- 金銭
- 削除
- 外部公開
- 個人情報
- セキュリティ境界
- 監査必須操作

# 第6章 状態・セッション・記憶

## 6.1 用語を分ける

```text
コンテキスト = 今回モデルへ見せる情報
状態 = 現在のタスク進行状況
セッション = 会話・イベント・状態の継続単位
記憶 = 将来のタスクで再利用する情報
成果物 = ファイル、計画、ログ、評価結果などの保存対象
```

## 6.2 正本(Source of Truth)を用途別に分ける

イベントログは監査・再生へ有効ですが、常に業務上の正本にする必要はありません。次を分けます。

| 対象 | 代表的な正本 |
|---|---|
| 業務データ | 業務Database、外部System、Version管理されたArtifact |
| 現在のAgent状態 | State Store、Workflow Engine、Checkpoint |
| 監査・再生 | 追記専用イベントログ、Trace、CI Artifact |
| モデルへ渡す情報 | 上記から派生したContext、要約、Retrieved documents |
| 長期記憶 | 検証・期限・Tenant Scopeを伴うMemory Store |

推奨される関係は次です。

```text
業務データ / 現在状態 / 実行記録
→ 検証可能な派生Context
→ 要約
→ 記憶候補
→ 検証・昇格
→ 長期記憶
```

イベントログを再生の正本にする場合は、次を満たすか確認します。

- Event schemaをVersion管理できる
- Duplicateと順序逆転を扱える
- 個人情報の削除・Retention要件と両立できる
- Snapshotと再生結果の整合性を検証できる
- 業務DBとの二重正本を発生させない
- 保存量と再生時間を運用できる

モデルが作った要約だけを、現在状態や業務事実の正本にしません。

## 6.3 保存すべき状態

- task_id
- current_phase
- completed_items
- pending_items
- decisions
- changed_resources
- evidence_refs
- approvals
- budget_usage
- retry_counts
- last_checkpoint
- stop_reason

## 6.4 チェックポイント(Checkpoint)

チェックポイントには、再開に必要な最小情報を含めます。

- 現在の目的
- 完了条件
- 現在の状態
- 未完了項目
- 直前の成功操作
- 次に予定する操作
- 必要な出典・成果物参照
- 使用済み予算
- 有効な承認

## 6.5 記憶の種類

| 種類 | 内容 |
|---|---|
| 作業記憶(Working Memory) | 現在の計画、直近結果、未完了項目 |
| エピソード記憶(Episodic Memory) | 過去の実行、事故、対応履歴 |
| 意味記憶(Semantic Memory) | プロジェクト事実、判断、選好 |
| 手続き記憶(Procedural Memory) | 成功した手順、検証方法、運用手順 |

## 6.6 記憶へ昇格する条件

保存候補が次を満たすか確認します。

- 将来再利用する可能性がある
- 根拠がある
- 権威性が分かる
- 有効期間を定義できる
- 他情報との競合を管理できる
- 個人情報・機密情報の方針に適合する

保存しないもの：

- 一時的な雑談
- 未確認の推測
- 大きなツール結果全文
- 期限切れ状態
- 根拠のないユーザー属性
- 機密情報の平文

## 6.7 検索取得

推奨順序：

```text
スコープ・メタデータ絞込み
→ 完全一致検索
→ 意味検索
→ 再順位付け
→ 鮮度・権威性確認
→ 必要量だけコンテキストへ注入
```

意味検索だけに依存しないようにします。ファイル名、エラー文字列、識別子、日付などは完全一致検索が強い場合があります。

## 6.8 記憶の更新と削除

記憶には次を持たせます。

- valid_from
- valid_until
- authority
- confidence
- supersedes
- source_refs
- deletion_policy

古い記憶を上書きするのではなく、置換関係を残すと監査しやすくなります。

# 第7章 検証・評価・レビュー

## 7.1 3つを分ける

| 概念 | 問い |
|---|---|
| 検証(Verification) | 今回の処理・変更が要求どおり成立したか |
| 評価(Evaluation) | エージェントシステムの品質が基準を満たすか |
| レビュー(Review) | 変更が妥当、安全、保守可能か |

## 7.2 検証ループ

```text
実行
→ 構造検証
→ 値・意味の妥当性確認
→ 事後条件確認
→ 完了条件との照合
→ 成功 / 修正 / 再試行 / 人間へ引継ぎ
```

後段へ未検証データを流さないことが重要です。

## 7.3 決定論的検証を先に使う

優先順序：

1. スキーマ検証
2. 型検査
3. 静的解析
4. 単体テスト
5. 統合テスト
6. 業務規則
7. LLM評価
8. 人間評価

機械的に判定できるものを、LLMへ評価させる必要はありません。

## 7.4 評価の4層

### 構成要素評価(Component Eval)

- ツール選択
- 引数精度
- 検索結果
- 構造化出力
- 記憶検索

### 軌跡評価(Trajectory Eval)

- 不要なステップがないか
- 同じ操作を反復していないか
- 計画から逸脱していないか
- 適切な時点で人間へ引き継いだか

### 成果評価(Outcome Eval)

- タスクを完了したか
- 要求を満たしたか
- 副作用が正しいか
- 証拠があるか

### 運用評価(Operational Eval)

- 遅延
- コスト
- 再試行率
- 人間介入率
- エラー回復率
- 安全違反率

## 7.5 評価データセット

評価ケースには次を含めます。

- 正常系
- 境界値
- 失敗系
- 権限不足
- データ欠損
- 古い記憶
- ツールタイムアウト
- プロンプトインジェクション
- 人間承認が必要なケース
- 停止すべきケース

本番障害は、再発防止のため評価ケースへ追加します。

## 7.6 LLMを評価者に使う場合

LLMによる評価(LLM-as-a-Judge)は、大量評価や曖昧な文章評価に使えますが、絶対的な正解ではありません。

対策：

- 評価基準をルーブリック化する
- 人間が作った基準ケースで校正する
- 順序バイアスを減らす
- 長い回答を好む傾向を確認する
- 低信頼・不一致ケースを人間へ回す
- 評価モデルと生成モデルを分ける
- 評価プロンプトのバージョンを保存する

## 7.7 リリース判定の例

```yaml
# 例: 説明用の仮のリリースゲート。各指標は自社データで決める。
release_gate:
  min_task_success_rate: 0.92
  max_safety_violation_rate: 0.001
  max_tool_argument_error_rate: 0.02
  max_p95_latency_ms: 8000
  max_cost_per_success_usd: 0.40
  max_regression_cases: 0
```

特定の1指標だけで決めず、品質、安全性、費用、遅延を同時に見ます。

## 7.8 変更時に再評価する対象

- モデル
- プロンプト
- ツール定義
- ツール実装
- 検索方式
- 記憶方式
- コンテキスト圧縮
- 再試行
- ルーティング
- 権限ポリシー
- 完了判定

# 第8章 可観測性・トレース・再生

## 8.1 最終出力だけでは足りない

AIエージェントでは、HTTP 200が返っていても意味的には失敗している場合があります。

- 誤ったツールを使った
- 誤った引数を渡した
- 古い記憶を取得した
- 同じ手順を反復した
- 途中で計画を放棄した
- 予算を浪費した

可観測性(Observability)は、最終結果に至る過程を再構成できる状態です。

## 8.2 トレース階層

```text
session
└─ task
   └─ turn
      └─ step
         ├─ model_call
         ├─ retrieval
         ├─ memory_read
         ├─ tool_call
         ├─ verification
         ├─ approval
         └─ state_transition
```

## 8.3 最小トレース項目

- trace_id
- session_id
- task_id
- parent_span_id
- event_type
- timestamp
- model
- prompt_version
- tool_name
- tool_arguments_redacted
- result_status
- latency_ms
- input_tokens
- output_tokens
- estimated_cost
- retry_count
- state_before
- state_after
- evidence_refs
- error_code
- stop_reason

## 8.4 収集する指標

### 成果

- タスク成功率
- 完了条件達成率
- 証拠不足率
- 人間による差戻し率

### 軌跡

- 平均ステップ数
- 不要ツール呼出し率
- 反復率
- 計画逸脱率
- サブエージェント引継ぎ失敗率

### ツール

- 成功率
- 引数不正率
- 再試行率
- タイムアウト率
- P50 / P95 / P99遅延

### コスト

- タスク当たり費用
- 成功タスク当たり費用
- 機能別費用
- 顧客・テナント別費用
- 失敗へ使った費用

### 人間参加

- 承認要求率
- 承認率
- 差戻し率
- 上書き率
- 判断待ち時間

## 8.5 再生(Replay)

追記専用イベントから、過去の実行状態を再構成できるようにします。

再生用途：

- 障害調査
- 評価ケース生成
- UI表示
- 監査
- コスト分析
- 新しいモデルでの再実行
- 回帰確認

生ログを後から書き換えず、派生ビューを作る方が安全です。

## 8.6 プライバシー

可観測性のために、機密情報を無制限に保存してはいけません。

- 入力・出力のマスキング
- APIキー・トークンの除外
- 個人情報の削除・仮名化
- 保存期間
- アクセス制御
- テナント分離
- ログエクスポート監査

## 8.7 アラートの例

```yaml
# 例: 初期監視用の仮値。通常時の分布を測定してから調整する。
alerts:
  task_success_rate_drop_percentage_points: 5
  p95_latency_increase_ratio: 1.5
  cost_per_success_increase_ratio: 1.4
  repeated_action_rate_threshold: 0.08
  unsafe_tool_attempts_threshold: 1
```

# 第9章 信頼性・コスト・性能・回復

## 9.1 信頼性を構成する要素

- タイムアウト
- 再試行
- フォールバック(Fallback)
- サーキットブレーカー(Circuit Breaker)
- 冪等性
- チェックポイント
- ロールバック
- 補償処理(Compensation)
- 負荷制御
- コスト上限
- 機能縮退(Graceful Degradation)

## 9.2 タイムアウトは工程別に設定する

```text
全体タイムアウト
├─ モデル
├─ 検索
├─ ツール
├─ 承認待ち
└─ 検証
```

1つの全体タイムアウトだけでは、どこが遅いか分かりません。

## 9.3 フォールバック(Fallback)

代替モデル・代替ツールへ切り替える前に、次を確認します。

- 機能が同等か
- 出力スキーマが同じか
- 安全性が同等か
- データ所在地が許容されるか
- コストが許容されるか
- 高リスク操作を継続してよいか

高リスク操作では、精度が保証できない代替へ自動切替するより、機能を停止する方が安全な場合があります。

## 9.4 タスク別コスト上限(Cost Envelope)

タスク当たりの費用上限を持ちます。

決め方：

1. 代表タスクを実行する
2. 成功・失敗別に費用分布を測る
3. 外れ値の原因を分類する
4. タスク種別ごとに上限を設ける
5. 上限超過時の停止・引継ぎを定義する

```yaml
# 例: 説明用の仮値。実測した費用分布に置き換える。
cost_envelope:
  simple_lookup_usd: 0.05
  document_analysis_usd: 0.50
  code_change_usd: 2.00
  high_risk_review_usd: 3.00
  on_exceed: escalate
```

## 9.5 成功当たりコスト

単なるAPI料金ではなく、次を見ます。

```text
成功当たりコスト
= 全実行費用
÷ 成功タスク数
```

安いモデルが失敗・再試行を増やすと、成功当たりコストは高くなる場合があります。

## 9.6 遅延の分解

```text
総遅延
= キュー待ち
+ コンテキスト構築
+ モデル初回応答
+ 生成
+ ツール待ち
+ 検証
+ 人間待ち
```

追跡候補：

- TTFT
- TPOT
- ツールP50 / P95 / P99
- キュー待ち
- 再試行時間
- コンテキスト構築時間
- 承認待ち時間

## 9.7 サーキットブレーカーの例

```yaml
# 例: 説明用の仮値。サービス特性とSLOに合わせて調整する。
circuit_breaker:
  failure_window_seconds: 60
  open_after_failures: 5
  half_open_after_seconds: 30
  trial_requests: 2
```

## 9.8 機能縮退(Graceful Degradation)

障害時に、すべてを継続する必要はありません。

例：

- 書込みを停止し読取りだけ継続
- 大型モデルが使えないときは要約のみ提供
- 検索が使えない場合は「出典付き回答」を停止
- 承認基盤が停止した場合は高リスク操作を拒否
- 記憶が利用できない場合は現在セッションだけで動く

# 第10章 セキュリティと人間参加型制御

## 10.1 すべての外部情報を信頼しない

次はすべて非信頼入力(Untrusted Input)として扱います。

- ユーザー入力
- Webページ
- メール
- 添付ファイル
- RAG文書
- ツール結果
- 他エージェントの出力
- 記憶検索結果

文章中に命令が書かれていても、それをシステム命令として扱わない構造が必要です。

## 10.2 プロンプトインジェクションへの基本姿勢

目標は「モデルが絶対に騙されない」ことではありません。

```text
モデルが悪意ある指示を読む
→ 操作を提案する可能性がある
→ しかし権限層が拒否する
→ 事故にならない
```

## 10.3 最小権限(Least Privilege)

- 読取りと書込みを分ける
- 必要なパスだけ許可する
- 必要なAPIだけ公開する
- テナントを分離する
- 秘密情報をモデルへ直接渡さない
- 一時認証情報を使う
- 権限に期限を設ける
- 権限取消しを可能にする

## 10.4 サンドボックス(Sandbox)

プロンプトだけで危険操作を防がないようにします。

制限候補：

- ファイルシステム
- ネットワーク
- プロセス
- CPU・メモリ
- 実行時間
- 環境変数
- 認証情報
- デバイス
- 外部接続先

## 10.5 リスク分類

| リスク | 例 | 人間承認 |
|---|---|---|
| 低 | 読取り、要約、候補作成 | 通常不要 |
| 中 | 下書き、可逆な設定変更 | 条件付き |
| 高 | 外部送信、本番変更、個人情報利用 | 原則必要 |
| 重大 | 送金、大量削除、権限付与、法的確定 | 必須・複数統制も検討 |

## 10.6 意味のある承認

承認画面には次を含めます。

- 何を行うか
- なぜ必要か
- 対象
- 差分
- 代替案
- リスク
- 不可逆点
- 費用
- ロールバック方法
- 承認の有効範囲

悪い承認：

```text
この操作を許可しますか？
```

良い承認：

```text
顧客3件へ通知メールを送信します。
本文、宛先、送信理由、取消不能であることを表示し、
送信前の最終確認を求めます。
```

## 10.7 承認疲れを防ぐ

- リスクベースで承認する
- 低リスク操作は限定範囲でまとめる
- 差分を短く提示する
- 高影響項目を強調する
- 同一承認の有効期限を設ける
- 上書き率・差戻し率を測る
- 承認者が判断できない情報量にしない

## 10.8 脅威モデルを明示する

Prompt Injectionだけでなく、Agentが持つTool、Identity、Memory、外部接続、Supply Chainまで脅威モデルへ含めます。

| 脅威 | 代表的な失敗 | 主な制御 |
|---|---|---|
| データ流出 | Context、Tool引数、URL、最終出力から秘密が外部へ出る | Data classification、Egress制御、Redaction、Output validation |
| SSRF | Agentが内部Metadata ServiceやPrivate endpointへアクセスする | URL allowlist、DNS/IP検証、Network namespace、Proxy policy |
| Cross-tenant漏えい | 別User・別TenantのMemoryやRAG文書を取得する | Tenant-bound authorization、Row-level security、Cache key分離 |
| 悪意あるTool・MCP・Skill | Tool descriptionやPackageが権限を悪用する | 署名・Source review、Version pin、Tool allowlist、Sandbox |
| Supply Chain | 依存Package、Container、Plugin更新から侵害される | Lockfile、Artifact署名、SBOM、Dependency review、隔離Build |
| Memory / RAG汚染 | 悪意ある指示や誤情報が永続化する | Source provenance、昇格審査、TTL、Tenant分離、再検証 |
| Log / Trace漏えい | Prompt、PII、Token、Credentialが観測基盤へ残る | Opt-in content logging、Masking、Retention、Access control |
| Approvalすり替え | 承認後にTool引数・対象Versionが変わる | 正規化Action hash、Expiry、Replay protection、実行直前再検証 |
| 間接Prompt Injection | Web、Email、Tool resultにある命令をSystem命令として扱う | Instruction/Data分離、Tool authorization、Untrusted label |
| 権限昇格 | 低権限Agentが上位CredentialやToolへ到達する | Capability isolation、短期Credential、Policy service |
| Sandbox回避 | Shell、Symlink、Mount、Kernel経由で境界を越える | Hardened runtime、Patch、No-new-privileges、Defense in depth |
| 連鎖障害 | 1 Agentの侵害が他Agentへ伝播する | Message schema、Trust boundary、Circuit breaker、Quarantine |

OWASPのAI Agent Security Cheat SheetとAgentic Applications Top 10を、脅威の網羅性を確認する外部基準として利用します。ただし、自組織の資産、Actor、Data Flow、Trust Boundaryに合わせたThreat Modelを別途作成します。

# 第11章 マルチエージェント統合運用

## 11.1 分離する正当な理由

サブエージェントを使う理由は、次のいずれかです。

- コンテキストを分離する
- 並列実行する
- 異なる権限を与える
- 異なるモデルを使う
- 専門ツールを分ける
- 独立レビューを行う
- 長時間作業を別実行単位にする

「役割名を付けると賢く見える」は理由になりません。

## 11.2 単一エージェントを維持すべき条件

- タスクが短い
- 状態が単純
- 同じファイル・対象を扱う
- 並列化の利益が小さい
- 統合コストが高い
- 独立権限が不要
- 1つのコンテキストで十分

## 11.3 推奨パターン

### 調査 → 実装 → 独立レビュー

```text
調査役
→ 実装役
→ 新しいコンテキストのレビュアー
→ 必要なら実装役へ差戻し
```

### 並列調査

情報源や仮説を分け、親エージェントが統合します。

### 敵対的レビュー

作者の前提を継承しないレビュー役が、失敗条件、境界値、不要な複雑性を探します。

## 11.4 引継ぎ契約(Handoff Contract)

子へ渡すもの：

- 目的
- 完了条件
- 対象範囲
- 必要な出典
- 使用可能ツール
- 禁止操作
- 出力スキーマ
- 予算

渡さないもの：

- 親の全会話
- 無関係な推測
- 不要な認証情報
- 他の子の未検証結果
- 全リソースへの書込み権限

## 11.5 境界検証

サブエージェント出力を親へ渡す前に確認します。

- 必須項目
- 出典
- 空値
- 対象範囲
- 変更ファイル
- 検証結果
- 未解決事項
- 予算使用量
- 停止理由

## 11.6 起動予算(Spawn Budget)

```yaml
# 例: 小規模な並列調査向けの仮値。推奨値ではない。
orchestration_budget:
  max_active_agents: 3
  max_total_spawns_per_task: 8
  max_depth: 2
  max_wall_time_minutes: 20
  max_total_cost_usd: 4.00
  require_structured_result: true
```

同時稼働数と、タスク全体で起動できる総数を分けます。

## 11.7 アンチパターン

- 全エージェントへ同じ長文コンテキストを渡す
- 全エージェントへ書込み権限を与える
- 同じ対象を同時編集する
- レビュアーが作者の推論を完全継承する
- 子が無制限に子を作る
- 自由文だけで引継ぐ
- 失敗を多数決で無視する
- 親が待つだけで他の仕事をしない
- コスト上限がない

# 第12章 導入順序と本番移行

## 12.1 最小実用ハーネス(Minimum Viable Harness)

最初に必要なもの：

- 単一エージェントループ
- 少数の明確なツール
- 入出力スキーマ検証
- 実行予算
- 構造化エラー
- セッションログ
- 完了条件
- 証拠付き完了報告

最初から不要なもの：

- 大規模マルチエージェント
- 長期記憶
- 自己改善
- 複雑な動的プラグイン
- 分散実行
- 多数のモデルルーター

## 12.2 成熟段階

### 段階0　生成機能

- 1回のモデル呼出し
- 外部副作用なし
- 人間が全出力を確認

### 段階1　ツール利用

- 入出力検証
- 実行予算
- 構造化エラー
- 基本ログ

### 段階2　本番単一エージェント

- 状態永続化
- 回復
- 冪等性
- 評価
- 可観測性
- コスト制御
- 権限

### 段階3　高リスク・長時間運用

- 人間承認
- チェックポイント再開
- 監査
- サンドボックス
- 機能縮退
- オンライン評価

### 段階4　マルチエージェント

- 引継ぎ契約
- 親子トレース
- 起動予算
- 独立権限
- 統合責任

## 12.3 実装優先順位

一般的には次の順序が安全です。

```text
1. 完了条件
2. ツール入出力検証
3. 実行予算と停止条件
4. イベントログ
5. 評価ケース
6. 権限と冪等性
7. 状態永続化・再開
8. コスト・遅延最適化
9. 記憶
10. マルチエージェント
```

## 12.4 失敗モードから逆算する

機能を追加する前に、代表タスクを実行し、失敗を分類します。

| 主な失敗 | 先に追加するもの |
|---|---|
| ツール結果が壊れる | 出力検証 |
| 同じ操作を繰り返す | 進展判定・停止制御 |
| 古い情報を使う | 鮮度・出典管理 |
| 長時間で目的を失う | 外部状態・チェックポイント |
| コストが暴走する | 実行予算・タスク別コスト上限 |
| 原因が分からない | トレース |
| 間違った操作を実行する | 権限・人間承認 |
| 子エージェント間で壊れる | 引継ぎ検証 |

## 12.5 削除可能性レビュー

定期的に次を確認します。

- 古いモデル固有ルールを削除できるか
- 重複した検証を統合できるか
- 不要な再試行を削除できるか
- ツール定義を減らせるか
- 記憶を持たずに解決できるか
- マルチエージェントを単一化できるか
- 使用されていないフックや設定が残っていないか

## 12.6 本番移行判定

- [ ] 代表タスクの評価ケースがある
- [ ] 失敗系と停止系を検証した
- [ ] 高リスク操作を分類した
- [ ] コスト上限がある
- [ ] 追跡可能なtrace_idがある
- [ ] 冪等性・重複防止がある
- [ ] 人間承認が必要な地点を定義した
- [ ] ロールバックまたは補償処理を確認した
- [ ] 機密情報のログ方針がある
- [ ] 障害時の機能縮退を確認した
- [ ] 本番失敗を評価へ戻す運用がある
- [ ] 運用責任者と停止権限者が決まっている

---

<a id="chapter-13"></a>

# 第13章 並行実行・一貫性・不確定結果

## 13.1 `success`と`failure`だけでは足りない

外部副作用を伴うAgentでは、TimeoutやProcess Crashの時点で結果を断定できないことがあります。

```text
Request送信
→ 外部Systemで処理成功
→ Response受信前にTimeout
→ Agent側は成功か失敗か不明
```

状態候補を明示します。

| 状態 | 意味 |
|---|---|
| `succeeded` | 事後条件と証拠を確認できた |
| `failed` | 副作用が発生していない、または失敗を確認できた |
| `uncertain` | 外部で成功した可能性があり、再送してよいか判断できない |
| `reconciling` | 外部照会・監査Log・Webhookで結果を確認中 |
| `compensating` | 成功済み操作の取消し・補償を実施中 |
| `needs_human` | 自動判定できず、人間判断が必要 |

`uncertain`を単純な`failed`へ変換して再試行すると、二重送信・二重決済・重複作成につながります。

## 13.2 At-least-onceと冪等性

分散処理では、TaskやMessageが少なくとも1回配信され、重複する前提で設計する場合があります。

- 副作用ToolへIdempotency Keyを渡す
- 同じKeyの結果を保存し、再実行時に照合する
- KeyをTask ID、Tool、正規化引数、対象Versionへ結び付ける
- ProviderがIdempotencyを支援しない場合、事前予約Recordまたは重複検出を用意する
- `exactly once`という表現を、Transaction境界を説明せずに使用しない

## 13.3 重複取得を防ぐLease

複数Workerが同一Taskを取得する場合、Task Ownerと有効期限を保存します。

```text
worker_id
lease_until
task_version
heartbeat_at
```

Lease失効後の再取得では、前Workerの外部副作用が残っていないかをReconciliationしてから継続します。

## 13.4 楽観的ロックとCompare-and-Swap

複数Agentや人間が同じ状態を更新する場合、読取り時のVersionを実行時に再確認します。

```sql
UPDATE tasks
SET state = :next_state, version = version + 1
WHERE task_id = :task_id
  AND version = :expected_version;
```

更新件数が0なら、前提が古いため再計画または人間確認へ進みます。

## 13.5 外部副作用と保存の境界

次の2つを1つのLocal Transactionにできない場合があります。

```text
外部APIへ送信
Local DBへ結果保存
```

対策候補：

- Transactional Outbox / Inbox
- ProviderのIdempotency Key
- External Request IDの保存
- WebhookまたはStatus APIによる照会
- Reconciliation Job
- 補償処理(Compensating Action)

Outbox / Inboxは万能ではありません。外部Providerの実行保証と、自Systemの再送保証を分けて記述します。

## 13.6 承認対象を固定する

承認後に対象が変更されるTime-of-check to time-of-use問題を防ぎます。

承認Recordへ含める候補：

- actor
- tool_name
- normalized_arguments
- target_resource_id
- expected_resource_version
- action_hash
- issued_at / expires_at
- policy_version
- approver

実行直前にAction Hash、対象Version、承認期限、権限を再検証します。

## 13.7 Multi-Agentの同時更新

並列Writerを許可する場合：

- File、Record、DomainごとにOwnerを分ける
- Shared generated fileを避ける
- Public contractを先に固定する
- Base Versionを引継ぎ契約へ含める
- Merge後に全体検証する
- ConflictをModel任せで黙って解消しない

## 13.8 再開時の判断

Checkpointには「何を試みたか」だけでなく、外部処理の確認情報を含めます。

```text
last_committed_step
pending_external_request_ids
uncertain_actions
idempotency_keys
expected_versions
active_leases
valid_approvals
next_reconciliation_action
```

# 付録A 判断基準早見表

## A.1 エージェントにするか、決定論的ワークフローにするか

| 条件 | 決定論的ワークフロー | AIエージェント |
|---|---:|---:|
| 手順が固定 | 適する | 過剰になりやすい |
| 入力形式が安定 | 適する | 必須ではない |
| 曖昧な自然言語理解が必要 | 補助的 | 適する |
| 手順を事前に列挙できない | 不向き | 適する |
| 誤り許容度が極めて低い | 適する | 強い統制が必要 |
| 外部情報を探索する | 条件付き | 適する |
| 複数の代替経路がある | 実装量が増える | 適する |

判断原則：

> ルールで十分に書ける部分はコードで書き、曖昧性が価値を生む部分だけをモデルへ渡す。

## A.2 Skill・Tool・Hook・Subagentの使い分け

| 必要なもの | 選択 |
|---|---|
| 作業手順や専門知識 | スキル(Skill) |
| 外部状態の読取り・変更 | ツール(Tool) |
| 必ず実行したい決定論的処理 | フック(Hook) |
| コンテキスト・権限・モデルを分離 | サブエージェント(Subagent) |
| 外部ツールとの共通接続 | MCP |

## A.3 コンテキスト・状態・記憶・RAGの使い分け

| 情報 | 保存先 |
|---|---|
| 今回だけ必要 | コンテキスト |
| 現在の進行状況 | 状態 |
| 将来再利用する判断・選好 | 記憶 |
| 更新される外部知識 | RAG・検索 |
| 監査・再生の正本 | イベントログ |
| 大きな成果物・全文 | 外部ストレージ |

## A.4 Retry・Fallback・Escalate・Abortの使い分け

| 状況 | 対応 |
|---|---|
| 一時的失敗で同じ処理が有効 | Retry |
| 別サービス・別モデルで同等品質が可能 | Fallback |
| 価値判断、権限、不可逆操作が必要 | Escalate |
| 安全性や整合性を保証できない | Abort |

## A.5 人間承認が必要か

次のいずれかに該当する場合、承認を検討します。

- 不可逆
- 外部公開
- 金銭
- 権限変更
- 個人情報
- 法的効果
- 大量処理
- 予算超過
- 仕様上の価値判断
- ロールバック困難

## A.6 単一エージェントかマルチエージェントか

| 条件 | 単一 | 複数 |
|---|---:|---:|
| 同じコンテキストで完結 | 適する | 不要 |
| 並列調査で時間短縮 | 条件付き | 適する |
| 独立レビューが必要 | 条件付き | 適する |
| 権限を分ける必要 | 不向き | 適する |
| 同じファイルを編集 | 適する | 競合しやすい |
| 統合コストが高い | 適する | 不向き |

---

<a id="appendix-b"></a>

# 付録B 実装テンプレート集

> **この付録の全テンプレートは例です。**  
> フィールド名、数値、リスク区分、予算、閾値は、そのまま本番へ適用せず、対象業務のリスク評価・代表タスクの実測・SLO・法務要件・運用体制に合わせて変更してください。

## B.1 タスク契約(Task Contract)

```markdown
# タスク契約

## 目的

<!-- 何を解決するか -->

## 期待する成果

<!-- 作業ではなく成立してほしい状態 -->

## 完了条件

- [ ]
- [ ]

## 変更可能範囲

- 

## 禁止事項

- 

## 必要な証拠

- 実行コマンド:
- テスト結果:
- 差分:
- スクリーンショットまたはAPI結果:

## 人間判断が必要な事項

- 

## 停止条件

- 

## 残存リスクの報告形式

- 
```

## B.2 実行予算(Execution Budget)

```yaml
# 例: 説明用の初期値。代表タスクの実測後に調整する。
execution_budget:
  max_steps: 12
  max_tool_calls: 24
  max_wall_time_seconds: 180
  max_input_tokens: 120000
  max_output_tokens: 16000
  max_cost_usd: 1.00
  same_action_limit: 2
  same_error_limit: 2
  no_progress_limit: 3
  on_exceed: escalate
```

## B.3 コンテキスト方針(Context Policy)

```yaml
# 例: 数値は仮の配分。利用モデルとタスクに合わせて変更する。
context_policy:
  priority_order:
    - system_and_security
    - task_and_completion_criteria
    - active_state
    - relevant_sources
    - recent_tool_results
    - recent_history
    - retrieved_memory

  token_budget:
    system_and_security: 6000
    task_and_completion_criteria: 6000
    active_state: 5000
    relevant_sources: 42000
    recent_tool_results: 18000
    recent_history: 18000
    retrieved_memory: 8000
    answer_reserve: 20000

  compaction:
    preserve:
      - original_request
      - completion_criteria
      - pending_items
      - decisions
      - changed_resources
      - verification_results
      - evidence_refs
    offload_large_tool_results: true
```

## B.4 ツール定義(Tool Definition)

```yaml
name: send_notification
purpose: "承認済みの通知を指定された宛先へ送信する"
use_when:
  - "通知本文と宛先が確定している"
do_not_use_when:
  - "下書きだけを作る場合"
  - "送信承認がない場合"

risk_level: high
side_effect: external_message
requires_human_approval: true
requires_idempotency_key: true

input_schema:
  type: object
  required:
    - recipient_id
    - body
    - idempotency_key
  properties:
    recipient_id:
      type: string
    body:
      type: string
    idempotency_key:
      type: string

output_schema:
  type: object
  required:
    - message_id
    - sent_at
    - recipient_id
```

## B.5 ツール結果(Tool Result Envelope)

```json
{
  "ok": false,
  "status": "failed",
  "data": null,
  "evidence_refs": [],
  "side_effects": [],
  "retryable": true,
  "error_code": "RATE_LIMITED",
  "message": "Temporary rate limit",
  "details_redacted": {}
}
```

## B.6 状態チェックポイント(State Checkpoint)

```json
{
  "task_id": "task_123",
  "phase": "verify",
  "completion_criteria": [],
  "completed_items": [],
  "pending_items": [],
  "decisions": [],
  "changed_resources": [],
  "evidence_refs": [],
  "approval_refs": [],
  "budget_usage": {
    "steps": 0,
    "tool_calls": 0,
    "input_tokens": 0,
    "output_tokens": 0,
    "estimated_cost_usd": 0
  },
  "last_successful_event_id": "event_456",
  "next_recommended_action": null
}
```

## B.7 記憶レコード(Memory Record)

```json
{
  "memory_id": "mem_123",
  "scope": "project",
  "type": "decision",
  "subject": "authentication",
  "content": "採用した方針を記載する",
  "source_refs": ["session_1:event_24"],
  "authority": "user",
  "confidence": 1.0,
  "valid_from": "2026-08-26",
  "valid_until": null,
  "supersedes": [],
  "contains_sensitive_data": false
}
```

## B.8 評価ケース(Eval Case)

```yaml
id: expired_token_is_rejected
category: regression

input:
  request: "期限切れトークンで更新を試みる"

preconditions:
  token_status: expired

expected:
  task_completed: true
  response_status: 401
  database_write_count: 0
  unsafe_tool_calls: 0

# 例: 遅延・費用の値は説明用。実測したSLOに置き換える。
operating_envelope:
  max_latency_ms: 3000
  max_cost_usd: 0.05
  max_steps: 6

required_evidence:
  - response_log
  - database_audit
```

## B.9 トレースイベント(Trace Event)

```json
{
  "trace_id": "trace_123",
  "span_id": "span_456",
  "parent_span_id": "span_100",
  "session_id": "session_1",
  "task_id": "task_1",
  "event_type": "tool.completed",
  "timestamp": "2026-08-26T00:00:00Z",
  "tool_name": "example_tool",
  "status": "completed",
  "latency_ms": 0,
  "retry_count": 0,
  "input_tokens": 0,
  "output_tokens": 0,
  "estimated_cost_usd": 0,
  "evidence_refs": [],
  "error_code": null,
  "attributes_redacted": {}
}
```

## B.10 人間承認依頼(Human Approval Request)

```markdown
# 承認依頼

## 実行する操作

## 対象

## 実行理由

## 変更差分

## 代替案

## 主なリスク

## 不可逆な点

## 費用・数量

## ロールバックまたは補償方法

## 承認の有効範囲

- 対象:
- 回数:
- 有効期限:

## 選択肢

- [ ] 承認
- [ ] 条件付き承認
- [ ] 差戻し
- [ ] 拒否
```

## B.11 サブエージェント引継ぎ(Handoff Contract)

```yaml
role: reviewer
objective: "変更が要求、テスト、安全性、単純性を満たすか独立確認する"

scope:
  read:
    - "src/"
    - "tests/"
  write: []

tools:
  allowed:
    - read
    - search
    - test
  denied:
    - deploy
    - publish

required_output:
  status: "completed | blocked | failed"
  findings:
    - severity
    - location
    - evidence
    - recommendation
  verification:
    - command
    - result
  unresolved_questions: []
```

## B.12 完了報告(Completion Report)

```markdown
# 完了報告

## 実施内容

## 完了条件

- [x]
- [x]

## 証拠

### 実行コマンド

### 結果

### 変更対象

## 予算使用量

- ステップ:
- ツール呼出し:
- 推定費用:
- 実行時間:

## 残存リスク

## 未実施事項

## ロールバック方法
```

## B.13 インシデント記録(Incident Record)

```markdown
# AIエージェント・インシデント記録

## 概要

## 影響

## 検知方法

## タイムライン

## 直接原因

## 根本原因

## 失敗した境界

- [ ] コンテキスト
- [ ] ツール入力
- [ ] ツール出力
- [ ] 権限
- [ ] 状態
- [ ] 記憶
- [ ] 引継ぎ
- [ ] 完了判定
- [ ] 人間承認

## 封じ込め

## 復旧

## 再発防止

## 追加した評価ケース

## 削除・簡素化できる制御
```

---

# 付録D 共通用語集

本付録を、本シリーズの共通用語の正本とします。活用ハーネスは同じ定義を要約して再掲します。

| 用語 | 本シリーズでの意味 |
|---|---|
| ハーネス(Harness) | モデルの周囲で、コンテキスト、ツール、権限、状態、検証、評価、可観測性、コスト、回復を管理する実行・運用層 |
| タスク契約(Task Contract) | 目的、期待状態、範囲、禁止事項、完了条件、必要証拠、停止条件を開始前に固定した契約 |
| 完了条件(Completion Criteria) | タスクを終了してよいと判定する、観測可能な条件 |
| 証拠(Evidence) | 完了条件を満たしたことを確認するための、コマンド結果、差分、ログ、テスト、観測記録など |
| 実行予算(Execution Budget) | Step、時間、Tool Call、費用、再試行、並列数などの実行上限 |
| 検証(Verification) | 特定の出力・変更・事後条件が期待どおりかを確認する処理 |
| 評価(Evaluation) | 代表タスク集合を使い、システム品質を比較・測定する処理 |
| レビュー(Review) | 設計、差分、証拠、残存リスクを別の視点で吟味する行為 |
| 状態(State) | 現在の進行段階、完了項目、未完了項目、承認、予算使用量など、再開に必要な情報 |
| コンテキスト(Context) | そのModel Callでモデルへ実際に提示する情報 |
| 記憶(Memory) | 将来のタスクで再利用するため、検証・期限・スコープを伴って保存した情報 |
| セッション(Session) | 会話、イベント、状態、成果物を継続的に関連付ける実行単位 |
| 引継ぎ(Handoff) | 別の人・Agent・Sessionへ、目的、状態、制約、証拠、次の操作を構造化して渡すこと |
| 人間ゲート(Human Gate) | 人間の判断または承認がなければ次へ進めない境界 |
| 停止条件(Stop Condition) | 完了、予算超過、同一失敗、権限不足、不確定結果など、実行を止める条件 |
| 完了報告(Completion Report) | 実施内容、完了条件、証拠、未実施事項、残存リスク、Rollbackをまとめた最終成果物 |

### 補足

- `Verification`は個別成果の確認、`Evaluation`は代表Dataset上の比較測定、`Review`は別視点からの吟味です。
- `Context`と`Memory`を同一視しません。Memoryは保存層、ContextはそのCallへ提示する選択結果です。
- `Evidence`と`Completion Report`を同一視しません。Reportは証拠への索引であり、証拠そのものではありません。
- `Human Gate`はApprovalだけではなく、仕様決定、例外受入れ、Risk Acceptanceも含みます。

---

<a id="appendix-e"></a>

# 付録E 最小参照実装

`examples/minimal_agent_runtime.py`は、Provider非依存の最小Runtime例です。

含むもの：

- Task Contract
- Execution Budgetの強制
- Tool入力・出力Validation
- Tool allowlist
- 高リスクToolのAction hash付きApprovalと承認状態の記録
- Fileへ保存するIdempotency Store
- Checkpoint保存と再開
- `uncertain`を含むStop Reason
- Completion CriteriaとEvidenceの照合
- 同一Tool Call・同一Errorの停止

> **例:** 学習用の最小構成です。Production向けのAuthentication、暗号化、Durable Queue、Database Transaction、Distributed Lock、Telemetry Exporter、Secret管理を省略しています。数値も説明用です。

実行例：

```bash
cd examples
python -m unittest -v
```

```python
"""Provider-neutral minimum harness for a tool-using AI agent.

All budgets are examples. This omits production authentication, encryption,
distributed locking, queues, and telemetry exporters.
"""
from __future__ import annotations

import hashlib
import json
import time
from dataclasses import asdict, dataclass, field
from enum import Enum
from pathlib import Path
from typing import Any, Callable, Protocol


class RiskLevel(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"


class StopReason(str, Enum):
    COMPLETED = "completed"
    MAX_STEPS = "max_steps_exceeded"
    TIMEOUT = "timeout"
    DUPLICATE_ACTION = "duplicate_action"
    REPEATED_ERROR = "repeated_error"
    INVALID_ACTION = "invalid_action"
    APPROVAL_DENIED = "approval_denied"
    UNCERTAIN = "uncertain_external_result"


@dataclass(frozen=True)
class ExecutionBudget:
    # Example values only; replace them using representative-task measurements.
    max_steps: int = 8
    max_elapsed_seconds: float = 30.0
    same_action_limit: int = 2
    same_error_limit: int = 2


@dataclass(frozen=True)
class TaskContract:
    task_id: str
    objective: str
    completion_criteria: tuple[str, ...]
    allowed_tools: frozenset[str]
    workspace: Path


@dataclass(frozen=True)
class Action:
    kind: str  # "tool" or "complete"
    tool_name: str | None = None
    arguments: dict[str, Any] = field(default_factory=dict)
    summary: str | None = None
    evidence_refs: tuple[str, ...] = ()


@dataclass(frozen=True)
class ToolResult:
    status: str  # "success", "error", or "uncertain"
    output: dict[str, Any] = field(default_factory=dict)
    error_type: str | None = None
    external_request_id: str | None = None


@dataclass(frozen=True)
class ToolSpec:
    name: str
    risk: RiskLevel
    validate_input: Callable[[dict[str, Any]], dict[str, Any]]
    execute: Callable[[dict[str, Any]], ToolResult]
    validate_output: Callable[[ToolResult], ToolResult]


@dataclass
class RunState:
    task_id: str
    started_at_epoch: float = field(default_factory=time.time)
    phase: str = "running"
    step: int = 0
    observations: list[dict[str, Any]] = field(default_factory=list)
    evidence_refs: list[str] = field(default_factory=list)
    last_error_type: str | None = None
    same_error_count: int = 0
    action_counts: dict[str, int] = field(default_factory=dict)


@dataclass(frozen=True)
class RunResult:
    stop_reason: StopReason
    summary: str
    state: RunState


class DecisionProvider(Protocol):
    def decide(self, task: TaskContract, state: RunState) -> Action: ...


def stable_hash(value: Any) -> str:
    normalized = json.dumps(value, ensure_ascii=False, sort_keys=True, separators=(",", ":"))
    return hashlib.sha256(normalized.encode("utf-8")).hexdigest()


class JsonFile:
    def __init__(self, path: Path) -> None:
        self.path = path

    def read(self) -> dict[str, Any]:
        if not self.path.exists():
            return {}
        return json.loads(self.path.read_text(encoding="utf-8"))

    def write(self, value: dict[str, Any]) -> None:
        self.path.parent.mkdir(parents=True, exist_ok=True)
        temporary = self.path.with_suffix(self.path.suffix + ".tmp")
        temporary.write_text(json.dumps(value, ensure_ascii=False, indent=2), encoding="utf-8")
        temporary.replace(self.path)


class IdempotencyStore:
    """Persists successful or uncertain results so retries do not repeat side effects."""

    def __init__(self, path: Path) -> None:
        self.file = JsonFile(path)

    def get(self, key: str) -> ToolResult | None:
        raw = self.file.read().get(key)
        return ToolResult(**raw) if raw else None

    def put(self, key: str, result: ToolResult) -> None:
        if result.status not in {"success", "uncertain"}:
            return
        data = self.file.read()
        data[key] = asdict(result)
        self.file.write(data)


class CheckpointStore:
    def __init__(self, path: Path) -> None:
        self.file = JsonFile(path)

    def save(self, state: RunState) -> None:
        self.file.write(asdict(state))

    def load(self, task_id: str) -> RunState | None:
        raw = self.file.read()
        if not raw or raw.get("task_id") != task_id:
            return None
        return RunState(**raw)


class ApprovalService:
    """Binds approval to the exact normalized tool action."""

    def __init__(self, approve: Callable[[str, str], bool]) -> None:
        self._approve = approve

    def authorize(self, tool: ToolSpec, args: dict[str, Any]) -> tuple[bool, str]:
        digest = stable_hash({"tool": tool.name, "args": args})
        return (True, digest) if tool.risk is not RiskLevel.HIGH else (self._approve(tool.name, digest), digest)


class AgentRuntime:
    def __init__(self, tools: list[ToolSpec], budget: ExecutionBudget,
                 approvals: ApprovalService, idempotency: IdempotencyStore,
                 checkpoints: CheckpointStore) -> None:
        self.tools = {tool.name: tool for tool in tools}
        self.budget = budget
        self.approvals = approvals
        self.idempotency = idempotency
        self.checkpoints = checkpoints

    def run(self, task: TaskContract, model: DecisionProvider, *, resume: bool = False) -> RunResult:
        state = self.checkpoints.load(task.task_id) if resume else None
        state = state or RunState(task_id=task.task_id)
        state.phase = "running"

        while True:
            if state.step >= self.budget.max_steps:
                return self._stop(state, StopReason.MAX_STEPS, "step budget exhausted")
            if time.time() - state.started_at_epoch > self.budget.max_elapsed_seconds:
                return self._stop(state, StopReason.TIMEOUT, "elapsed-time budget exhausted")

            try:
                action = model.decide(task, state)
            except Exception as exc:  # Adapter failures become structured stop results.
                state.observations.append({"type": "decision_error", "message": str(exc)})
                return self._stop(state, StopReason.INVALID_ACTION, "decision provider failed")
            state.step += 1

            if action.kind == "complete":
                available = set(state.evidence_refs) | set(action.evidence_refs)
                missing = [criterion for criterion in task.completion_criteria if criterion not in available]
                if missing:
                    state.observations.append({"type": "completion_rejected", "missing_evidence": missing})
                    self.checkpoints.save(state)
                    continue
                state.evidence_refs = sorted(available)
                return self._stop(state, StopReason.COMPLETED, action.summary or "completed")

            if action.kind != "tool" or not action.tool_name:
                return self._stop(state, StopReason.INVALID_ACTION, "unsupported action")
            tool = self.tools.get(action.tool_name)
            if tool is None or tool.name not in task.allowed_tools:
                return self._stop(state, StopReason.INVALID_ACTION, "tool unavailable or outside scope")

            try:
                args = tool.validate_input(action.arguments)
            except (TypeError, ValueError) as exc:
                if self._record_error(state, "validation_error", str(exc)):
                    return self._stop(state, StopReason.REPEATED_ERROR, "repeated validation error")
                self.checkpoints.save(state)
                continue

            action_key = stable_hash({"tool": tool.name, "args": args})
            state.action_counts[action_key] = state.action_counts.get(action_key, 0) + 1
            if state.action_counts[action_key] > self.budget.same_action_limit:
                return self._stop(state, StopReason.DUPLICATE_ACTION, "same tool call repeated")

            state.phase = "awaiting_approval"
            approved, action_digest = self.approvals.authorize(tool, args)
            state.observations.append({"type": "approval", "action_digest": action_digest, "approved": approved})
            self.checkpoints.save(state)
            if not approved:
                return self._stop(state, StopReason.APPROVAL_DENIED, "approval was denied")
            state.phase = "running"

            idem_key = stable_hash({"task": task.task_id, "tool": tool.name, "args": args})
            try:
                result = self.idempotency.get(idem_key) or tool.execute(args)
                result = tool.validate_output(result)
            except (TypeError, ValueError) as exc:
                if self._record_error(state, "invalid_tool_output", str(exc)):
                    return self._stop(state, StopReason.REPEATED_ERROR, "repeated invalid tool output")
                self.checkpoints.save(state)
                continue

            self.idempotency.put(idem_key, result)
            state.observations.append({"tool": tool.name, "args": args, "result": asdict(result)})
            if result.status == "uncertain":
                return self._stop(
                    state, StopReason.UNCERTAIN,
                    f"reconcile external request before retry: {result.external_request_id}",
                )
            if result.status == "error":
                if self._record_error(state, result.error_type or "tool_error", "tool failed"):
                    return self._stop(state, StopReason.REPEATED_ERROR, "same tool error repeated")
            else:
                state.last_error_type, state.same_error_count = None, 0
                state.evidence_refs.extend(map(str, result.output.get("evidence_refs", [])))
            self.checkpoints.save(state)

    def _stop(self, state: RunState, reason: StopReason, summary: str) -> RunResult:
        state.phase = "completed" if reason is StopReason.COMPLETED else "stopped"
        self.checkpoints.save(state)
        return RunResult(reason, summary, state)

    def _record_error(self, state: RunState, error_type: str, message: str) -> bool:
        state.same_error_count = state.same_error_count + 1 if state.last_error_type == error_type else 1
        state.last_error_type = error_type
        state.observations.append({"type": error_type, "message": message})
        return state.same_error_count >= self.budget.same_error_limit


def validate_read_args(args: dict[str, Any]) -> dict[str, Any]:
    path = args.get("path")
    if not isinstance(path, str) or not path:
        raise ValueError("path must be a non-empty string")
    return {"path": path}


def validate_tool_result(result: ToolResult) -> ToolResult:
    if not isinstance(result, ToolResult) or result.status not in {"success", "error", "uncertain"}:
        raise ValueError("invalid tool result")
    if result.status == "uncertain" and not result.external_request_id:
        raise ValueError("uncertain result requires external_request_id")
    return result


def make_read_tool(workspace: Path) -> ToolSpec:
    root = workspace.resolve()

    def execute(args: dict[str, Any]) -> ToolResult:
        target = (root / args["path"]).resolve()
        if target != root and root not in target.parents:
            return ToolResult(status="error", error_type="path_outside_workspace")
        if not target.is_file():
            return ToolResult(status="error", error_type="file_not_found")
        return ToolResult("success", {"text": target.read_text(encoding="utf-8"),
                                      "evidence_refs": [str(target)]})

    return ToolSpec("read_file", RiskLevel.LOW, validate_read_args, execute, validate_tool_result)
```

---

<a id="appendix-f"></a>

# 付録F 主要主張と参考根拠の対応

| 本書の主張・章 | 主な外部根拠 | 本書での使い方 |
|---|---|---|
| AIリスクをLifecycle全体で継続管理する | NIST AI RMF Core / Playbook | 第1章、第7章、第9章、第12章のGovern・Measure・Manageの基礎 |
| 生成AI固有のリスクを既存Risk Managementへ追加する | NIST AI 600-1 Generative AI Profile | Context、評価、Monitoring、Incident、Third-party Riskの補助 |
| Tool最小権限、Memory隔離、HITL、Multi-Agent Security | OWASP AI Agent Security Cheat Sheet | 第5章、第6章、第10章、第11章、第13章のSecurity requirement |
| Goal hijack、Tool misuse、Identity abuse、Agentic supply chain、Unexpected code execution | OWASP Top 10 for Agentic Applications | 第10章のThreat ModelとSecurity Test Case |
| LLM / GenAI Applicationの主要Risk。Prompt Injection、Sensitive Information Disclosure、Supply Chain、Excessive Agency、Unbounded Consumptionなどの版管理されたRisk分類 | OWASP GenAI LLM Top 10 2026、および必要に応じて2025版とのCrosswalk | 第3章、第5章、第9章、第10章の失敗モード。Risk IDと名称は参照Versionを固定 |
| Trace / Metric / Logで共通命名を使う | OpenTelemetry Semantic Conventions | 第8章のAttribute設計。GenAI項目はStatusとVersionを確認して採用 |
| Model単体ではなくContext、Tool、Verification、Observabilityを含むHarnessを見る | Agent Harness公開記事、公開投稿群 | 中核原則の整理。個別数値は採用しない |
| Tool schema・停止理由・Rate limit・Data handlingはProviderごとに確認する | 採用Providerの公式文書 | 第3章、第5章、第8章、第9章の実装時の正本 |

この対応表は、各Frameworkをそのまま実装Checklistへ変換するものではありません。自SystemのContext、Risk Tolerance、法令、データ分類、SLOへTailorします。

---

<a id="appendix-g"></a>

# 付録G 参考文献

## NIST

- [AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [AI RMF Core](https://airc.nist.gov/airmf-resources/airmf/5-sec-core/)
- [NIST AI RMF Playbook](https://www.nist.gov/itl/ai-risk-management-framework/nist-ai-rmf-playbook)
- [Artificial Intelligence Risk Management Framework: Generative Artificial Intelligence Profile (NIST AI 600-1)](https://www.nist.gov/publications/artificial-intelligence-risk-management-framework-generative-artificial-intelligence)
- [NIST AI Resource Center](https://airc.nist.gov/)

> AI RMF 1.0とPlaybookは任意利用のFrameworkであり、2026年8月時点で改訂作業中です。適用時はCurrent versionを確認してください。

## OWASP

- [AI Agent Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html)
- [OWASP Top 10 for Agentic Applications for 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)
- [OWASP GenAI LLM Top 10 2026](https://genai.owasp.org/resource/owasp-genai-llm-top-10-2026/)
- [OWASP Gen AI Security Project](https://genai.owasp.org/)
- [Insecure Agent Samples](https://genai.owasp.org/resource/insecure-agent-samples/)

## OpenTelemetry

- [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)
- [OpenTelemetry GenAI Semantic Conventions Repository](https://github.com/open-telemetry/semantic-conventions-genai)
- [GenAI Attribute Registry](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/)

> GenAI固有のSemantic ConventionsにはDevelopment状態の項目があります。Attribute名とSchema URLをVersion固定し、機密Contentの記録はOpt-inとします。

## Provider・Security Frameworkの例

- [Anthropic Tool Use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview)
- [Google Secure AI Framework (SAIF)](https://saif.google/)
- [Google SAIF Map](https://saif.google/secure-ai-framework)

採用ProviderのTool Calling、Structured Output、Stop Reason、Rate Limit、Data Retention、Authentication、Regionality、Safety、LoggingのCurrent docsを実装時の正本にしてください。

## Agent Harness参考情報

- [Agent Harness Home](https://agent-harness.ai/)
- [Harness Engineering: The 80% Factor in Agent Reliability](https://agent-harness.ai/blog/what-is-harness-engineering/)
- [AI Agent Monitoring: Tools, Metrics, and Best Practices](https://agent-harness.ai/blog/ai-agent-monitoring-tools-metrics-and-best-practices/)

Agent Harnessの記事に含まれる個別の改善率・閾値・比較値は、環境と評価方法へ依存するため、本書の推奨値としては使用していません。

---

# 最終要約

AIエージェント開発で重要なのは、モデルへ長い指示を与えることではありません。

```text
目的を定義する
→ 完了条件を定義する
→ 操作を制限する
→ 境界で検証する
→ 状態を保存する
→ 実行を観測する
→ 予算内で停止する
→ 証拠付きで完了する
→ 本番失敗を評価へ戻す
```

信頼できるAIエージェントとは、常に正しいエージェントではありません。

> **間違えたときに検知でき、被害を限定でき、原因を追跡でき、再発を防止できるエージェントシステム**です。
