---
title: "AI Harness 共通用語集"
version: "1.0"
status: "normative"
updated: "2026-09-02"
---

# AI Harness 共通用語集

本書をシリーズ全体の用語の正本とします。製品固有文書が別の意味で用語を使う場合は、製品固有語であることを明示します。

## 中核概念

| 用語 | 定義 |
|---|---|
| モデル（Model） | 入力から推論・生成・ツール候補の提案を行う構成要素。権限、状態、実行、監査の所有者ではない。 |
| エージェント（Agent） | モデル、コンテキスト、ツール、状態、実行ループ、権限、検証を組み合わせてタスクを進める実行主体。 |
| ハーネス（Harness） | モデルの周囲で、コンテキスト、ツール、権限、状態、検証、評価、可観測性、コスト、回復を管理する実行・運用層。 |
| Agent Product | エージェントを業務価値、UI、承認、データ、運用責任へ結び付けた製品全体。 |
| Vendor Harness | Codex等の製品が提供するAgent Loop、Tool execution、Session、承認UI、圧縮など。 |
| Project Harness | 利用者がRepository、Git、CI、Policy、Verificationとして所有する統制層。 |

## 契約と完了

| 用語 | 定義 |
|---|---|
| タスク契約（Task Contract） | 目的、期待状態、Scope、禁止事項、完了条件、必要証拠、能力、停止条件を実行前に固定するVersion付き契約。 |
| 契約改訂（Contract Amendment） | 調査中に前提・Scope・Risk・完了条件が変わった場合に、旧契約を保持したまま新Versionを有効化する手続き。 |
| Contract Hash | 正規化済みTask Contractから生成し、承認、実行、証拠、完了判断を同じ契約へ結び付けるHash。 |
| 完了条件（Completion Criterion） | Taskを完了としてよいか第三者が観測できる、識別子付きの条件。 |
| Verification Result | 一つのCompletion Criterionを、誰が、何で、どの対象Versionに対して検証したかを表す構造化結果。 |
| Evidence | Completion Criterionの成立を確認するためのTest、API応答、差分、Log、Screenshot、監査Event等。 |
| Evidence Binding | EvidenceをCriterion、対象Commit・Artifact・Action、Verifier、時刻へ結び付けること。 |
| Completion Decision | 必須Criterion、Evidence、Scope、Policy、未解決Error、残存Riskを統合した採用・拒否・保留の最終判断。 |
| Completion Report | 実施内容、Criterionごとの結果、Evidenceへの参照、未実施、残存Risk、Rollbackを人間向けにまとめた索引。 |

## リスク・権限・ポリシー

| 用語 | 定義 |
|---|---|
| Risk Assessment | 機密性、完全性、可用性、外部副作用、可逆性、金銭・法務、Credential、Tenant境界、検出可能性、復旧時間を個別に評価すること。 |
| Risk Band | 多軸評価を低・中・高・重大などへ要約した運用上の分類。Capabilityを直接意味しない。 |
| Capability Profile | File、Network、Tool、Credential、外部Sink、人間Gate等、実行主体へ与える能力の組合せ。 |
| Data Classification | Public、Internal、Confidential、Restricted等、入力・派生物の取扱いを決めるLabel。 |
| Sink | メール、Issue、URL、Log、外部API、公開Repositoryなど、データが書き込まれる先。 |
| Policy Decision Point（PDP） | 主体、契約、Resource、操作、データ分類、送信先、Version、承認、Policy Versionから認可判断を返す論理的な判定点。物理的に一つのServiceである必要はない。 |
| Policy Enforcement Point（PEP） | PDPの判定を、Tool gateway、OS、Network、DB、CI等で実際に強制する境界。 |
| Policy Decision | `allow`、`deny`、`require_approval`のいずれかと、理由、義務、Policy Versionを含む構造化結果。 |
| Human Gate | 人間の判断・承認がなければ次へ進めない境界。仕様判断、例外受入れ、Risk Acceptanceも含む。 |
| Approval Record | 正規化済み操作、対象Version、Action Hash、差分、送信先、不可逆性、承認者、期限を記録した承認。 |
| Action Hash | Tool名、正規化引数、対象Resource・Version、送信先等から作り、承認した操作と実行操作の一致を確認するHash。 |
| Break-glass | 通常Policyを緊急時に限定解除する手続き。理由、期限、権限、監査、事後Reviewが必須。 |

## 実行・状態・コンテキスト

| 用語 | 定義 |
|---|---|
| 実行予算（Execution Budget） | Step、時間、Tool Call、費用、再試行、変更量、並列数等の上限。 |
| 停止条件（Stop Condition） | 完了、予算超過、同一失敗、権限不足、不確定結果、Scope逸脱等、実行を止める条件。 |
| 状態（State） | 現在Phase、完了・未完了項目、承認、予算、再開位置等、Runtimeが進行判断に使う情報。 |
| コンテキスト（Context） | そのModel Callへ実際に提示する情報。保存層ではない。 |
| セッション（Session） | 会話、Event、状態、成果物を関連付ける実行単位。 |
| 記憶（Memory） | 将来のTaskで再利用するため、根拠、期限、Scope、Tenantを伴って保存した情報。 |
| Checkpoint | 中断後に安全に再開するための最小状態Snapshot。 |
| Handoff | 別の人、Agent、Session、Environmentへ、目的、状態、制約、Evidence、次Actionを構造化して渡すこと。 |
| Source of Truth | 特定の事実・状態について最終的に正しいと扱う正本。業務Data、Agent State、監査Log、モデル要約を分離する。 |

## 信頼性・分散実行

| 用語 | 定義 |
|---|---|
| 冪等性（Idempotency） | 同じ操作を複数回試みても、副作用が一度だけ成立するか、同一結果へ収束する性質。 |
| Idempotency Key | 同一操作を識別し、重複実行を検出するKey。Task、Tool、正規化引数、対象Versionへ結び付ける。 |
| Lease | Workerが一定期間Taskを所有する権利。有効期限だけでは古いWorkerの書込みを完全には防げない。 |
| Fencing Token | Lease取得ごとに単調増加し、保存先が古いWorkerの操作を拒否するためのToken。 |
| Compare-and-Swap（CAS） | 読み取ったVersionが変わっていない場合だけ更新する競合制御。 |
| Outbox / Inbox | Local Transactionと外部Messageの境界で、送信・受信の記録と重複処理を管理するPattern。 |
| Uncertain | Timeout等により、外部副作用が成功したか失敗したか断定できない状態。単純なFailureとして再送しない。 |
| Reconciliation | 外部Request ID、Status API、Webhook、監査Log等から、Uncertainな結果を照合する処理。 |
| Compensation | 既に成立した副作用を取り消す、または業務上相殺する処理。 |

## 再生・評価・レビュー

| 用語 | 定義 |
|---|---|
| Audit Replay | 保存Eventから過去の状態を再構築すること。外部Toolを再実行しない。 |
| Simulation Replay | 過去のTool応答を固定し、新ModelやHarnessだけを評価すること。副作用ToolはStub化する。 |
| Live Re-execution | 実際の外部Toolを再度呼び出す新しいRun。Replayとは分け、別承認と冪等性管理を要求する。 |
| Verification | 今回の処理・変更が特定のCriterionを満たしたか確認すること。 |
| Evaluation | Version付きDataset上でシステム品質を比較・測定すること。 |
| Review | 設計、差分、Evidence、残存Riskを別の視点から吟味すること。 |
| Reviewer Assurance | Context、Workspace、権限、Model、実行、専門性の各独立性を表すProfile。単純な一本のLevelだけでは表さない。 |
| Trusted CI | Writer Agentが結果や保存先を自由に変更できない環境で、固定Commitを独立検証するCI。 |
| Eval Contamination | 評価Case、期待結果、採点基準が生成側へ漏れ、見かけの性能を押し上げること。 |
| Drift | 本番Task分布、入力、Tool、Model、Policyが評価Datasetからずれていくこと。 |

## 関連文書

- [中核設計原則](principles.md)
- [統合判定モデル](decision-model.md)
