---
title: "Profile: 分散・長時間・複数Worker実行"
version: "1.0"
core_version: "1.0"
profile_id: "distributed-execution"
risk_scope: "medium-to-critical"
updated: "2026-09-02"
---

# Profile: 分散・長時間・複数Worker実行

## 目的

Queue、複数Worker、長時間Task、Process再起動、外部副作用を扱うときに必要な一貫性・重複防止・再開Controlを定義します。

単一Processで十分なTaskへ、この複雑性を追加しません。

---

## 適用Trigger

- Queue
- At-least-once Delivery
- Multiple Workers
- Worker Failover
- Long-running Task
- Checkpoint Resume
- Parallel Agent
- External Side Effect
- Local DBと外部SystemのTransaction分離
- Timeout後に外部成功の可能性
- Multiple Tenant / Shard
- Scheduled Reconciliation

---

## 状態

```text
queued
→ leased
→ running
→ verifying
→ succeeded
```

Failure / Uncertainty:

```text
running
├─ failed
├─ uncertain
├─ reconciling
├─ compensating
├─ needs_human
└─ cancelled
```

`success`と`failure`だけにしません。

---

## At-least-once

TaskやMessageは重複する前提で設計します。

必要:

- Stable Task ID
- Contract Hash
- Operation ID
- Idempotency Key
- Payload Hash
- Expected Version
- Result Store
- Duplicate Detection
- Audit

「Exactly once」と書く場合は、どのTransaction境界で成立するかを説明します。

---

## Idempotency

Key例:

```text
task_id
+ contract_version
+ tool_name
+ normalized_arguments_hash
+ target_resource
+ target_version
```

処理:

```text
Keyなし
→ reservation
→ execute
→ result保存

Keyあり / same payload
→ existing resultを返す

Keyあり / different payload
→ conflict、実行しない
```

外部ProviderがKeyを支援する場合も、自SystemのResult Storeと照合します。

---

## Lease

Record:

```yaml
task_id: TASK-001
lease_owner: worker-2
lease_until: "2026-09-02T12:01:00Z"
heartbeat_at: "2026-09-02T12:00:30Z"
task_version: 8
fencing_token: 42
```

Leaseは所有権の有効期限です。しかし、Network PartitionやPauseにより古いWorkerがLease失効後も動く可能性があります。

---

## Fencing Token

Lease取得ごとに単調増加Tokenを発行します。

```text
worker-1 gets token 41
→ lease expires
worker-2 gets token 42
→ storage accepts 42
worker-1 resumes with 41
→ storage rejects stale token
```

State Store、Tool Gateway、Resource Adapterは、最後に受理したTokenより小さいTokenを拒否します。

### 限界

外部ProviderがFencing Tokenを検証できない場合:

- Provider Idempotency Key
- External Request ID
- Reservation
- Single-writer Gateway
- Status API
- Webhook
- Reconciliation
- Compensation

を使い、完全に防げない範囲を明記します。

---

## Optimistic Lock / CAS

```sql
UPDATE tasks
SET state = :next_state,
    version = version + 1
WHERE task_id = :task_id
  AND version = :expected_version
  AND fencing_token <= :current_token;
```

更新0件:

- Current Stateを再読込み
- Contract / Version確認
- External Side Effect確認
- ReplanまたはStop

ModelにConflictを黙って解消させません。

---

## External Side EffectとLocal State

問題:

```text
External API succeeds
→ Process crashes
→ Local result not saved
```

または:

```text
Local pending saved
→ External API times out
→ success / failure unknown
```

対策:

- Transactional Outbox
- Inbox / Deduplication
- Provider Idempotency
- External Request ID
- Status API
- Webhook
- Reconciliation Job
- Compensation

OutboxだけでExternal ProviderのExactly-onceを保証できるとは書きません。

---

## Uncertain State

Timeout時:

```text
送信Request
→ Providerで成功
→ Response前にTimeout
```

Runtimeは次を保存します。

```yaml
status: uncertain
action_hash: "sha256:..."
idempotency_key: "..."
external_request_id: "provider-123"
requested_at: ""
target_resource: ""
target_version: ""
payload_hash: "sha256:..."
next_reconciliation_action: "status API"
```

`uncertain`を`failed`へ変えて再送しません。

---

## Reconciliation

情報源:

- Provider Status API
- External Request ID
- Webhook
- Audit Log
- Business State Query
- Inbox
- Human Confirmation

結果:

- resolved_success
- resolved_failure
- still_uncertain
- needs_human
- compensate

Reconciliation自体もIdempotentにし、Evidenceを残します。

---

## Checkpoint

必須候補:

- Active Contract Hash
- State Version
- Last committed step
- Pending Criteria
- Pending External Request IDs
- Uncertain Actions
- Idempotency Keys
- Expected Resource Versions
- Active Lease / Fencing Token
- Valid Approvals
- Budget Usage
- Next Reconciliation

再開時はCheckpointを盲信せず、Business Systemと外部Providerの現在状態を再確認します。

---

## Parallel Agent

Read並列:

- 同じSnapshot
- 狭いTask
- Structured Result
- Parent Integration
- Spawn Budget

Write並列:

- Resource / File Ownership
- Separate Worktree
- Base Version
- Public Contract固定
- Shared Generated File回避
- CAS
- Fencing
- Merge / Integration Owner
- Post-integration Verification

---

## Event Ordering

Eventへ持たせるもの:

- event_id
- task_id
- sequence or logical version
- occurred_at
- recorded_at
- producer
- schema_version
- idempotency_key
- causation_id
- correlation_id
- fencing_token
- payload_hash

重複、順序逆転、遅延到着、Schema Version差を扱います。

Current StateをEventから作る場合、SnapshotとReplay整合性を検証します。

---

## Replay

### Audit Replay

EventからState再構築。External Toolなし。

### Simulation Replay

過去Tool Resultを固定。Side Effect Stub。

### Live Re-execution

新Run。新Lease、Policy、Approval、Idempotency、External State確認。

Live Re-executionをRetryやAudit Replayと混同しません。

---

## Failure Injection

検証Case:

- Worker crash before / after external call
- Response timeout after provider success
- Lease expiry
- Old Worker resume
- Duplicate Queue delivery
- Out-of-order Event
- CAS Conflict
- DB write failure
- Webhook duplicate / loss
- Provider status unavailable
- Approval expiry during run
- Contract amendment during run
- Network partition
- Clock skew
- Poison message
- Reconciliation conflict

---

## Observability

Metrics:

- Queue latency
- Lease expiry
- Fencing rejection
- Duplicate delivery
- Idempotency hit / conflict
- Uncertain rate
- Reconciliation duration
- Compensation rate
- Stuck Task
- Checkpoint age
- Retry by error class
- External Provider latency
- Human escalation
- Cost per completed task

Alert:

- Uncertain backlog
- Reconciliation SLA breach
- Repeated stale token
- Duplicate side effect
- Lease churn
- Poison message
- Audit gap

---

## Stop Conditions

- FencingをResource側で強制できずRisk許容不能
- Idempotency不在
- External Result照会不能
- Uncertain Backlog上限
- State Version Conflict継続
- Lease Service障害
- Audit / Result Store障害
- Approval / Contract mismatch
- Compensation不能なCritical Action
- Cross-tenant mismatch
- Queue Poisoning
- Budget超過

---

## Profile適合条件

- At-least-onceと重複を前提にする
- Idempotency KeyとResult Store
- LeaseとHeartbeat
- Fencing Tokenを古いWorker拒否へ使用
- CAS / Version Check
- Outbox / Inboxの保証範囲を説明
- `uncertain`とReconciliation
- Checkpoint再開時に外部状態を再確認
- Audit / Simulation / Live Re-executionを分離
- Parallel WriterのOwnershipと統合責任
- Failure Injection
- Uncertain BacklogとFencing Rejectionを監視
- 複雑性が本当に必要か定期Review

## 関連文書

- [エージェントランタイム](../learning/intermediate/agent-runtime.md)
- [AIエージェント開発リファレンス](../reference/agent-development-handbook.md)
- [本番高リスクプロファイル](production-high-risk.md)
