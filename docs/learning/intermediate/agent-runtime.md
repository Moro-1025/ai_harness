---
title: "中級: AIエージェント・ランタイムの設計"
version: "1.2"
core_version: "1.0"
audience: "intermediate"
risk_scope: "low-to-high"
updated: "2026-09-02"
---

# AIエージェント・ランタイムの設計

## ゴール

単一のModel Callから、状態を持ち、Toolを使い、失敗時に停止・回復・引継ぎできるRuntimeへ拡張します。

```text
Task Contract
→ State Machine
→ Action Proposal
→ Policy / Validation
→ Tool Execution
→ Verification
→ State Transition
→ CompletionまたはStop
```

この文書では、Model SDK固有の実装ではなく、Providerに依存しない責任境界を扱います。

---

## 1. Runtimeの責任

Runtimeは少なくとも次を所有します。

- TaskとContract Versionの読込み
- Active State
- Context構築
- Model Call
- Tool候補の受領
- Action Validation
- Authorization
- Tool Execution
- Retry、Timeout、Budget
- Event記録
- Verification
- Checkpoint
- Stop Reason
- Completion Decisionへの入力

Modelへ任せてはいけないもの:

- 自分の権限付与
- Budgetの解除
- Approvalの作成
- Event Logの削除
- Evidenceの真正性判定
- Policy Versionの選択的無視
- 未実行CriterionのPass化

---

## 2. 状態機械

推奨する最小State:

```text
intake
→ clarify
→ plan
→ authorize
→ execute
→ verify
→ complete
```

Failure時:

```text
execute / verify
→ recover
→ authorize / execute
```

自動回復不能:

```text
any state
→ reconcile
→ needs_human
→ stopped
```

状態の責任:

| State | 責任 |
|---|---|
| intake | Contract、対象、Ownerを受け取る |
| clarify | 高影響の曖昧さを解消する |
| plan | Step、依存、Evidence、Budgetを決める |
| authorize | Capability、Policy、Approvalを確認する |
| execute | Toolを使って状態を観測・変更する |
| verify | Output、Post-condition、Criterionを確認する |
| recover | Retry、Fallback、Rollback、Replanを選ぶ |
| reconcile | 不確定な外部副作用を照合する |
| needs_human | 人間が判断できるDecision Packetを作る |
| complete | Completion DecisionとReportを確定する |
| stopped | Stop Reasonと再開条件を保存する |

State遷移は自由文ではなく、許可されたTransitionとして実装します。

---

## 3. Active State

例:

```json
{
  "task_id": "TASK-001",
  "contract_hash": "sha256:...",
  "phase": "verify",
  "completed_criteria": ["AUTH-001"],
  "pending_criteria": ["AUTH-002"],
  "changed_resources": ["src/auth/refresh.py"],
  "evidence_refs": ["artifact://run/123/auth-expired.xml"],
  "approval_refs": [],
  "budget_usage": {
    "steps": 7,
    "tool_calls": 5,
    "cost": 0.21
  },
  "retry_counts": {
    "test_auth_expired": 1
  },
  "uncertain_actions": [],
  "last_checkpoint": "checkpoint://run/123/7",
  "stop_reason": null
}
```

Current Stateの正本とEvent Logを分けます。

```text
State Store
  └─ 現在判断に必要な正規化状態

Append-only Event Log
  └─ 何が起きたかを監査・再構築する記録

Business System
  └─ 実際の顧客、決済、Repository、外部Resource

Model Context
  └─ 上記から今回必要な情報だけを派生
```

Modelが作った要約だけをState Storeへ書き込みません。

---

## 4. Context Builder

優先順位:

1. 安全・権限規則
2. Active ContractとCompletion Criteria
3. 現在Stateと未完了項目
4. 直近のVerification
5. 直接関係するCode・Data
6. 採用済み判断
7. Tool Resultの要約
8. 古い会話・探索Log

Contextへ入れる情報には可能な限り次を持たせます。

```yaml
source_id: ""
source_type: repository
retrieved_at: ""
resource_version: ""
authority: project
classification: internal
tenant: ""
expires_at: null
```

圧縮対象:

- 完了済み探索Log
- 大きなTool Result
- 重複Code
- 解決済みError
- 古いPlan

圧縮後も保持:

- 原要求
- Active Contract Hash
- Completion Criteria
- Pending Items
- 重要判断と理由
- Changed Resources
- Verification Results
- Current Blocker
- Evidence参照

---

## 5. Action Proposal

ModelからのOutputは、自然言語ではなく構造化します。

```json
{
  "action": "read_file",
  "arguments": {
    "path": "src/auth/refresh.py",
    "start_line": 1,
    "max_lines": 200
  },
  "reason": "期限切れTokenのError mappingを確認する",
  "expected_observation": "例外が500へ変換される経路",
  "completion_criteria": ["AUTH-001"]
}
```

`reason`は説明であり、認可根拠ではありません。

---

## 6. Tool実行Pipeline

```text
Tool名Allowlist
→ Input Schema
→ Path / Resource Scope
→ Tenant / Purpose
→ Authentication
→ Policy Decision
→ Approval
→ Idempotency
→ Tool Call
→ Output Schema
→ Semantic Validation
→ Post-condition
→ Evidence
→ Event
```

### Input Validation

- required
- type
- enum
- min / max
- length
- pattern
- allowed resource
- path scope
- tenant
- destructive flag
- expected version
- idempotency key

JSONが構文上正しくても、業務上正しいとは限りません。

### Output Validation

HTTP 200でも次はFailureです。

- 本文が空
- 必須Fieldがない
- 対象IDが違う
- 更新件数が想定外
- 時刻・Versionが古い
- 部分結果
- Errorが本文に埋まっている
- Post-conditionが成立していない

---

## 7. Tool Result Envelope

共通形式:

```json
{
  "ok": true,
  "status": "succeeded",
  "data": {},
  "subject": {
    "resource_id": "auth-policy",
    "resource_version": "42"
  },
  "evidence_refs": [],
  "side_effects": [],
  "retryable": false,
  "error_code": null,
  "message": null
}
```

Failure:

```json
{
  "ok": false,
  "status": "failed",
  "data": null,
  "subject": null,
  "evidence_refs": [],
  "side_effects": [],
  "retryable": true,
  "error_code": "RATE_LIMITED",
  "message": "一時的なRate Limit"
}
```

不確定:

```json
{
  "ok": false,
  "status": "uncertain",
  "data": null,
  "external_request_id": "provider-req-123",
  "idempotency_key": "task-1:send:42",
  "side_effects": [
    {
      "type": "message",
      "may_have_occurred": true
    }
  ],
  "retryable": false,
  "error_code": "RESPONSE_TIMEOUT"
}
```

`uncertain`は通常のRetry対象にしません。

---

## 8. Error分類

| 分類 | 例 | 基本対応 |
|---|---|---|
| transient | 429、一時的5xx、接続失敗 | Backoff付き限定Retry |
| invalid_input | Schema、型、必須不足 | 引数修正 |
| permission | 認可失敗 | StopまたはHuman |
| policy | 上限、Destination、分類違反 | Deny、代替案 |
| conflict | Version不一致、CAS失敗 | 再読込み、Replan |
| permanent | 対象なし、未対応 | 経路変更またはStop |
| uncertain | Timeout時に外部成功の可能性 | Reconciliation |
| unknown | 分類不能 | 安全側にStop |

Retry前に分類します。同じ引数を無条件で再送しません。

---

## 9. Execution BudgetとNo Progress

例:

```yaml
execution_budget:
  max_steps: 12
  max_tool_calls: 24
  max_wall_time_seconds: 300
  max_cost: 1.00
  same_action_limit: 2
  same_error_limit: 2
  no_progress_limit: 3
  on_exceed: escalate
```

値は説明用です。Task種別ごとの実測へ置き換えます。

No ProgressのSignal:

- 同じToolと引数
- 同じFileを往復
- 同じError Code
- State Hash不変
- Pending Criterionが減らない
- Evidenceが増えない
- 同じ仮説の言い換え
- Scope外変更だけ増える

No ProgressはModelの自己評価だけで判定しません。

---

## 10. Idempotency

副作用Toolは、再試行・再開・重複配信を前提にします。

```json
{
  "operation": "send_notification",
  "idempotency_key": "task-123:contract-2:step-7:recipient-456",
  "payload_hash": "sha256:...",
  "target_version": "42"
}
```

RuntimeはKeyと結果を保存します。

```text
初回
→ reservationを作成
→ 外部Tool
→ resultをKeyへ保存

再実行
→ 同じKeyを照会
→ payload hashが同じなら既存結果
→ 異なるならConflict
```

ProviderがIdempotencyを支援しない場合、事前予約、重複検出、外部結果照会、Compensationの限界を明記します。

---

## 11. Checkpointと再開

Checkpoint:

```yaml
task_id: TASK-001
contract_hash: "sha256:..."
phase: reconcile
last_committed_step: 7
pending_criteria:
  - SEND-002
pending_external_request_ids:
  - provider-req-123
uncertain_actions:
  - action_hash: "sha256:..."
idempotency_keys:
  - task-1:send:42
expected_versions:
  customer-record: "42"
valid_approvals:
  - approval://123
budget_usage:
  steps: 7
next_reconciliation_action: "provider status APIを照会"
```

再開時は、「最後に何を試みたか」だけでなく、外部で完了した可能性を確認します。

### 再開手順

1. Active ContractとPolicy Versionを確認
2. Approval期限・Action Hashを再確認
3. Lease / Ownerを取得
4. Uncertain ActionをReconcile
5. Business Systemの現在Versionを読む
6. Checkpointと現実の差を計算
7. Continue、Replan、Compensate、Humanを決定

---

## 12. Replayを分ける

### Audit Replay

EventからStateを再構築します。Toolを呼ばず、副作用なし。

用途:

- 障害調査
- UI再構築
- 監査
- Cost分析

### Simulation Replay

保存Tool ResultをStubとして、新ModelやHarnessを評価します。

用途:

- Regression
- Prompt変更比較
- Policy変更比較
- Failure Case再現

### Live Re-execution

外部Toolを再び呼ぶ新Runです。

- 新しいRun ID
- 現在Contract・Policy
- 再承認
- Idempotency
- External Request照会
- Audit

Live Re-executionをAudit Replayの一部として自動化しません。

---

## 13. 最小Loop

```python
def run(task, model):
    state = load_or_create_state(task)

    while True:
        enforce_budget(state)

        if completion_candidate(state):
            decision = evaluate_completion(state)
            if decision.status == "accepted":
                return decision

        proposal = model.propose(build_context(state))
        action = validate_and_authorize(proposal, state)

        if action.requires_approval:
            return create_decision_packet(state, action)

        result = execute_idempotently(action)
        record_event(state, action, result)

        if result.status == "uncertain":
            transition(state, "reconcile")
            checkpoint(state)
            continue

        verification = verify_result(action, result, state)
        record_verification(verification)
        update_state(state, action, result, verification)

        if verification.status in {"blocked", "waived"}:
            return escalate(state)

        if no_progress(state):
            return stop("no_progress", state)

        checkpoint_if_needed(state)
```

---

## 14. 分散実行へ進む条件

次の必要が確認されるまで単一Processを維持します。

- Queueによる負荷平準化
- 長時間Taskの再開
- 複数TenantのIsolation
- 複数WorkerのFailover
- 並列Read
- 異なる権限のWorker
- 独立Reviewer Process

分散化すると、Lease、Fencing Token、CAS、Outbox / Inbox、Event順序、Duplicate、Schema Versionが必要になります。[分散実行プロファイル](../../profiles/distributed-execution.md)を適用してください。

---

## 15. 完了条件

- RuntimeとModelの責任を分離した
- State Machineと許可Transitionがある
- Current State、Event Log、Business Data、Contextを分離した
- Tool Input / Output / Post-conditionを検証する
- PolicyとApprovalを実行前に確認する
- Errorを分類してからRetryする
- BudgetとNo Progressで停止する
- 副作用ToolにIdempotencyを持たせる
- `uncertain`とReconciliationを実装する
- Checkpointから現実を再確認して再開する
- Audit、Simulation、Live Re-executionを分離する
- 分散化を観測済みの必要性が出るまで遅らせる

## 参照

- [統合判定モデル](../../core/decision-model.md)
- [Task Contract Schema](../../core/schemas/task-contract.schema.json)
- [Verification Result Schema](../../core/schemas/verification-result.schema.json)
- [AIエージェント開発リファレンス](../../reference/agent-development-handbook.md)
- [分散実行プロファイル](../../profiles/distributed-execution.md)
