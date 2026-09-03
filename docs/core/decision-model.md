---
title: "AI Harness 統合判定モデル"
version: "1.0"
status: "normative"
updated: "2026-09-02"
---

# AI Harness 統合判定モデル

本書は、個別に存在するTask Contract、Risk、Capability、Policy、Execution、Verification、Evidence、Human Gateを、一つの判定フローへ統合します。

```text
Task Contract
→ Risk / Data Classification
→ Capability Profile
→ Authorization
→ Approval（必要時）
→ Execution
→ CriterionごとのVerification
→ Evidence Binding
→ Completion Decision
→ Audit / Reconciliation
```

この順序は、AIエージェント開発とAIコーディングエージェント利用の両方に適用します。

---

## 1. Task Contractを固定する

Taskを開始する前に、少なくとも次をVersion付きで定義します。

- 目的と期待状態
- 読取り・書込みScope
- Non-goalと禁止事項
- 識別子付きCompletion Criteria
- CriterionごとのEvidence要件
- Risk Assessment
- Capability Profile
- Human Gate
- Execution Budget
- Stop Conditions

Task Contractは正規化し、`contract_hash`を生成します。承認、Policy Decision、Tool execution、Verification、Completion Decisionは同じHashを参照します。

### 契約の改訂

調査中に前提、Scope、Risk、完了条件が変わった場合、暗黙に広げません。

```text
変更候補を検知
→ 現Runを安全な地点で停止
→ Amendmentを作成
→ RiskとCapabilityを再評価
→ 必要な再承認
→ 新VersionとHashを有効化
→ 旧Evidence・旧承認の有効性を再判定
```

最低限の改訂情報は次です。

```yaml
contract_version: 2
previous_contract_hash: "sha256:..."
change_reason: "調査の結果、公開API変更が必要と判明"
changed_fields:
  - "scope.write"
  - "completion_criteria"
risk_delta: "integrity: medium -> high"
required_reapproval: true
```

旧契約は削除せず、`superseded`として保持します。

---

## 2. Risk AssessmentとCapability Profileを分ける

「低・中・高」だけで権限を決めません。Riskは少なくとも次の軸で評価します。

| 軸 | 問い |
|---|---|
| 機密性 | 読み取るだけでも漏えい時の影響が大きいか |
| 完全性 | 誤変更が業務・安全・監査へ与える影響は何か |
| 可用性 | 停止や過負荷がどこまで波及するか |
| 外部副作用 | メール、公開、決済、削除等が発生するか |
| 可逆性 | 自動Rollback、補償、人手復旧が可能か |
| 金銭・法務 | 支払い、契約、規制、説明責任へ影響するか |
| Credential | Secret、Token、権限昇格へ到達できるか |
| Tenant境界 | 別User・顧客Dataへ混線する可能性があるか |
| 検出可能性 | 誤りを独立して早期検知できるか |
| 復旧時間 | 許容時間内に復旧できるか |

Risk Bandは要約です。実際の能力はCapability Profileで定義します。

```yaml
risk_band: high

capability_profile:
  filesystem: read
  data_classifications:
    - internal
    - confidential
  network:
    mode: denied
  external_side_effects: denied
  credential_access: none
  allowed_tools:
    - repository_search
    - read_file
  human_gate: not_required
```

この例は「読取り専用」ですが、Confidential Dataを扱うためRisk Bandは高くなり得ます。

---

## 3. Data Classificationと情報フローを判定する

Tool単位のAllowlistだけでは、次の合成を防げません。

```text
機密情報を読むTool
→ モデルが要約
→ 外部メール・URL・Issue・Logへ書くTool
```

入力にはClassification、Tenant、Purpose、Originを付け、派生出力へLabelを伝播します。

### 原則

1. Modelが要約しただけではClassificationを自動で下げない。
2. Redaction後の再分類は、Modelと独立したDLPまたは決定論的検査を通す。
3. Sinkごとに許容Classification、Tenant、Purpose、Destinationを定義する。
4. Restricted Dataの読取り能力と外部書込み能力を、同一Sessionで同時に有効化しない構成を優先する。
5. Log、Trace、Error MessageもSinkとして扱う。
6. 外部ContentはUntrusted Inputとして扱い、命令とDataを分離する。

```yaml
data_flow:
  input_classification: confidential
  derived_classification: confidential
  tenant: tenant-a
  purpose: support-analysis
  destination:
    type: issue_tracker
    tenant: tenant-a
    maximum_classification: internal
  decision: deny
  reason: "派生出力の分類がSink上限を超える"
```

---

## 4. Authorizationを論理的なPDPへ集約する

認可判断は、次の入力を受け取ります。

```text
主体
+ Task Contract / Contract Hash
+ 対象Resource
+ 操作
+ Data Classification
+ 外部送信先
+ 対象Version
+ 有効な承認
+ Policy Version
→ allow / deny / require_approval
```

実装は一つのPolicy Serviceでも、OS、Tool Gateway、Network、DB、CIへ分散しても構いません。ただし、判定契約、優先順位、記録形式を共通化します。

### 判定優先順位

```text
Hard deny
> deny
> require_approval
> allow
```

- 一つでもHard Boundaryが拒否すれば実行しません。
- Policy評価に失敗した場合、高リスク操作はFail Closedします。
- 承認が必要な判定は、承認取得後に同じ入力Hashで再評価します。
- Contract、対象Version、Action、承認期限、Policy Versionが変わった場合は再評価します。
- `allow`は能力の付与ではありません。PEPが実際の境界を強制します。

Policy Decisionは[policy-decision.schema.json](schemas/policy-decision.schema.json)に従って保存します。

---

## 5. Human Gateを信頼境界として設計する

人間が画面を見るだけでは安全になりません。承認疲れ、専門性不足、役割衝突、要約による重要差分の埋没、承認後変更を失敗モードとして扱います。

承認UIは、Modelの自由文だけではなく、信頼できるSourceから次を生成します。

- 正規化済みTool引数
- 対象ResourceとVersion
- 実際の変更差分
- 外部送信先
- 件数・金額
- 不可逆性
- Action Hash
- Policy Decision
- Rollbackまたは補償方法
- 承認期限
- 必要な承認者Role

### 実行直前の再確認

```text
approval.action_hash == execution.action_hash
AND
approval.contract_hash == active_contract_hash
AND
approval.target_version == current_target_version
AND
approval is not expired / revoked / consumed
AND
current policy still allows
```

一致しなければ再承認します。高リスクProfileでは、職務分離、二者承認、専門家承認、Break-glass監査を追加します。

---

## 6. Executionを制御する

Tool実行は次のPipelineを通します。

```text
Action候補
→ 入力Schema
→ Authentication
→ Authorization
→ Data Flow Policy
→ Human Gate
→ Idempotency
→ 実行
→ 出力Schema
→ 事後条件
→ Evidence生成
→ Audit Event
```

### 状態

最低限、次を区別します。

| 状態 | 意味 |
|---|---|
| `succeeded` | 事後条件とEvidenceを確認した |
| `failed` | 副作用がない、または失敗を確認した |
| `uncertain` | 外部で成功した可能性があり再送可否を断定できない |
| `reconciling` | 外部照会・Webhook・監査Logで照合中 |
| `compensating` | 成功済み副作用の取消し・相殺中 |
| `needs_human` | 自動判断できず人間へ移譲した |

`uncertain`を`failed`へ変換して再試行してはなりません。

### 実行予算

Step、時間、Tool Call、費用、同一Action、同一Error、No Progress、変更File数、並列数を制限します。上限値は例をコピーせず、代表Taskの分布から設定します。

---

## 7. CriterionごとにVerificationを作る

`evidence_refs`の一括配列だけでは、どのEvidenceがどのCompletion Criterionを確認したか分かりません。

一つのCriterionごとに、[verification-result.schema.json](schemas/verification-result.schema.json)に従う結果を作ります。

```yaml
criterion_id: AUTH-001
assertion: "期限切れTokenで401を返す"
status: pass
verifier:
  type: integration-test
  name: auth-integration
  independent_from_writer: true
subject:
  repository: example/api
  commit: abc123
evidence_refs:
  - "ci://run/123/tests/auth-expired"
verified_at: "2026-09-02T12:00:00Z"
```

### Status

- `pass`: Criterionを満たすEvidenceがある
- `fail`: Criterionを満たさない
- `not_run`: 実行していない
- `blocked`: DependencyやEnvironmentにより確認不能
- `waived`: 権限を持つ人間が理由・期限・残存Riskを受け入れた

`not_run`を`pass`として扱いません。`waived`はApproval Recordなしでは有効にしません。

### Evidenceの評価軸

- 関連性
- 完全性
- 独立性
- 改ざん耐性
- 鮮度
- 対象固定
- 再現性
- 不確実性の明示

---

## 8. Completion Decisionを機械的に作る

完了は次の論理式で判断します。

```text
すべての必須Criterionに有効なVerificationがある
AND
Verificationが現在のContract Hashと対象Versionへ結び付いている
AND
必要なPolicy DecisionとApprovalが有効
AND
Scope逸脱がない
AND
重大な未解決Errorがない
AND
Uncertainな副作用が残っていない
AND
残存Riskが報告されている
```

出力は[completion-decision.schema.json](schemas/completion-decision.schema.json)に従います。

| Decision | 意味 |
|---|---|
| `accepted` | 採用条件を満たした |
| `rejected` | 明確なFailureまたはPolicy違反がある |
| `blocked` | 必要な検証・Dependency・権限が不足している |
| `needs_human` | Risk Acceptance、仕様判断、Waiver等が必要 |

Agent自身が候補を作ることはできますが、高リスクTaskの最終AcceptanceをAgentの自己申告だけで確定しません。

---

## 9. Replayを分離する

### Audit Replay

保存済みEventから過去Stateを再構築します。外部Toolは呼びません。

### Simulation Replay

過去のTool応答を固定し、新Model、Prompt、Harness、Policyを比較します。副作用ToolはStub化します。

### Live Re-execution

実際のTool・外部Serviceを再度呼び出す新しいRunです。

- 新しいTaskまたはRun IDを発行する
- 現在のContract、Policy、対象Versionを再評価する
- 別のApprovalを要求する
- Idempotency Keyと外部Request IDを確認する
- 二重送信・二重決済・重複作成を防ぐ

「Replay」という名前でLive Re-executionを自動実行してはなりません。

---

## 10. Reviewer Assuranceを多軸で表す

Reviewerの独立性は一本のLevelだけでなく、次の軸で記録します。

```yaml
reviewer_assurance:
  context: fresh
  workspace: clean_checkout
  permissions: read_only
  evaluator: different_model
  execution: trusted_ci
  expertise:
    - security
```

高リスクほど、異なる独立性を組み合わせます。別Modelだけで決定論的Testの代わりにはなりません。Trusted CIだけでRequirementやUXの人間判断の代わりにもなりません。

---

## 11. Schemaの関係

```text
task-contract.schema.json
  ├─ contract_hash
  ├─ completion_criteria[]
  ├─ risk_assessment
  └─ capability_profile

policy-decision.schema.json
  └─ contract_hash + action/input hash

approval-record.schema.json
  └─ contract_hash + action_hash + policy_decision_ref

verification-result.schema.json
  └─ contract_hash + criterion_id + subject + evidence_refs

completion-decision.schema.json
  └─ criterion_results + policy + approval + scope + unresolved risk
```

Schemaは形を検証します。意味の正しさ、Hashの計算、参照先の存在、承認者数の充足、Evidenceの真正性は、RuntimeまたはTrusted CIが検証します。

## 関連文書

- [中核設計原則](principles.md)
- [共通用語集](glossary.md)
- [証拠とポリシー](../learning/intermediate/evidence-and-policy.md)
- [本番高リスクプロファイル](../profiles/production-high-risk.md)
- [分散実行プロファイル](../profiles/distributed-execution.md)
