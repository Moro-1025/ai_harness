---
title: "AIエージェント開発リファレンス"
subtitle: "設計・実装・評価・運用の上級参照文書"
version: "1.2"
core_version: "1.0"
audience: "advanced-reference"
updated: "2026-09-02"
---

# AIエージェント開発リファレンス

> AIエージェントを「賢いモデルを呼び出す機能」ではなく、**失敗を検知し、安全に停止し、証拠を残しながら仕事を完了するソフトウェアシステム**として設計するための参照文書です。

中核原則、用語、機械可読Schemaは`docs/core/`を正本とします。本書は、原則を自作エージェントのArchitectureへ適用する方法を体系的に説明します。

旧v1.1全文は[ルートの旧版](../../AI_Agent_Development_Handbook_JA.md)に残しています。新規設計では本書とCoreを優先してください。

---

## 1. 読み方

| 目的 | 主な章 |
|---|---|
| 最小の単一Agentを作る | 2、3、4、5、7、14 |
| 本番運用へ進める | 6、8、9、10、12、13 |
| RAG・Memory | 4、6、10 |
| Multi-Agent | 3、11、12 |
| Security Review | 2、5、10、12 |
| 分散・長時間実行 | 6、9、13 |
| 実装前の標準確認 | [統合判定モデル](../core/decision-model.md) |

初めて実装する場合は、[読取り専用の単一エージェント](../learning/beginner/simple-agent-tutorial.md)から開始してください。

---

# 2. システム境界と責任分離

## 2.1 4層

```text
Model
  ↓
Runtime
  ↓
Harness
  ↓
Agent Product
```

| 層 | 主な責任 |
|---|---|
| Model | 意図理解、推論、生成、Action候補 |
| Runtime | Process、Queue、永続化、再開、同時実行 |
| Harness | Context、Tool、Permission、Policy、Verification、Eval、Observability、Cost |
| Product | 業務価値、UI、Human Gate、Business Data、監査、運用責任 |

一つのSDKやFrameworkが複数責任を実装していても、設計上の責任は分けます。

## 2.2 判断の配置

| 判断 | Model | Deterministic Code / Policy | Human |
|---|:---:|:---:|:---:|
| 自然言語の意図理解 | 主 | 補助 | 必要時 |
| Action候補 | 主 | Validation | 必要時 |
| Input型 | 不可 | 主 | 不要 |
| 認証・認可 | 不可 | 主 | 例外承認 |
| 金額・件数上限 | 不可 | 主 | 上限変更 |
| Data Flow | 候補 | 主 | 高Risk例外 |
| 外部送信内容 | 主 | DLP・Policy | 高Risk時 |
| 本番反映 | 提案 | 実行制御 | 主 |
| Completion | 候補 | Evidence照合 | 高Risk時 |
| Audit | 不可 | 主 | 監督 |

「Modelが判断できる」と「Modelへ最終権限を与える」は別です。

## 2.3 強いHarnessが必要な条件

- 外部API
- Data書込み
- 複数Stepの自律実行
- 長時間・再開
- 人間が毎回出力を確認しない
- Credential、個人情報、機密情報
- 金銭、権限、公開、削除
- 複数Tenant
- 複数Worker
- 説明責任・監査
- Failure時のCompensation

---

# 3. Task Contract、計画、Agent Loop

## 3.1 Task Contract

Task開始前に[Task Contract Schema](../core/schemas/task-contract.schema.json)で次を固定します。

- 目的と期待状態
- Scope、Forbidden Scope、Non-goal
- Constraint
- Completion Criteria
- Required Evidence
- Risk Assessment
- Capability Profile
- Human Gate
- Execution Budget
- Stop Condition
- Version、Contract Hash

Contractを固定する目的は、Modelを縛ることだけではありません。Policy、Approval、Verification、Acceptanceの基準を同じ対象へ結び付けるためです。

## 3.2 Contract Amendment

調査で前提が変わった場合:

```text
Stop
→ Amendment
→ Risk再評価
→ Capability再選択
→ 必要なApproval
→ 新Contract Hash
→ Evidenceの再利用可否
→ Resume
```

旧契約と旧承認を上書きしません。

## 3.3 Plan

複雑Taskでは、PlanをModel内部の一時的思考ではなくArtifactにします。

- 現状理解
- 変更対象 / 非対象
- Step
- Dependency
- StepごとのVerification
- Rollback / Compensation
- Human Gate
- Open Question
- Contract変更条件

## 3.4 Agent Loop

```python
def run(task, model):
    state = load_state(task)

    while True:
        enforce_budget(state)

        if completion_candidate(state):
            decision = evaluate_completion(state)
            if decision.accepted:
                return decision

        proposal = model.propose(build_context(state))
        action = validate_authorize_and_bind(proposal, state)

        if action.requires_approval:
            return create_approval_packet(action, state)

        result = execute_idempotently(action)
        append_event(action, result)

        if result.status == "uncertain":
            transition(state, "reconcile")
            checkpoint(state)
            continue

        verification = verify(action, result, state)
        persist_verification(verification)
        update_state(state, verification)

        if no_progress(state):
            return stop("no_progress", state)
```

## 3.5 Stop Reason

例:

```text
completed
budget_exceeded
no_progress
permission_required
policy_denied
approval_required
verification_failed
scope_boundary_exceeded
dependency_unavailable
uncertain_external_result
user_cancelled
unrecoverable_error
```

自由文だけでなく機械集計できるCodeを持たせます。

---

# 4. Context、State、Session、Memory

## 4.1 分離

```text
Context = 今回Modelへ見せる情報
State = 現在のTask進行
Session = 会話・Event・Stateの継続単位
Memory = 将来のTaskで再利用する情報
Artifact = File、Plan、Log、Test、Patch等
```

## 4.2 Source of Truth

| 対象 | 代表的な正本 |
|---|---|
| Business Data | 業務DB、外部System、Version Artifact |
| Current Agent State | State Store、Workflow Engine |
| Audit / Replay | Append-only Event、Trace、CI Artifact |
| Model Context | 正本から派生した情報 |
| Long-term Memory | Provenance、TTL、Tenant付きMemory Store |

Model要約をCurrent StateやBusiness Factの正本にしません。

## 4.3 Context Policy

優先:

1. Safety / Permission
2. Active Contract / Criteria
3. Current State / Pending
4. Direct Source
5. Recent Verification
6. Adopted Decision
7. Recent Tool Result
8. Old Conversation

ContextにはSource、Version、Authority、Classification、Tenant、Freshnessを可能な限り付けます。

## 4.4 Compaction

外部保存:

- Tool Result全文
- 大量Log
- Binary
- Source全文
- Raw Transcript

圧縮後も保持:

- Original Request
- Contract Hash
- Criteria
- Pending
- Important Decisions
- Changed Resources
- Verification
- Current Blocker
- Evidence References

## 4.5 Memory

Memoryへ昇格する前に確認:

- 将来再利用する価値
- SourceとAuthority
- Validity
- Tenant / Purpose
- Conflict管理
- Secret / PII方針
- 削除可能性

未確認の推測、一時状態、大きなTool Result全文、CredentialをMemoryにしません。

---

# 5. Tool設計、Policy、情報フロー

## 5.1 良いTool

- 1 Tool 1責任
- Nameから用途が分かる
- Use / Do-not-use条件
- 狭いInput
- Structured Error
- Side Effect明示
- Result Size制限
- Idempotency
- Subject Version
- Audit可能

## 5.2 実行Pipeline

```text
Action
→ Input Schema
→ Authentication
→ Authorization
→ Business Rule
→ Data Flow Policy
→ Human Gate
→ Idempotency
→ Execute
→ Output Schema
→ Semantic Validation
→ Post-condition
→ Evidence
→ Audit
```

## 5.3 CapabilityとPolicy

Risk AssessmentからCapability Profileを導出し、個別ActionはPDPで判断します。

```text
Subject
+ Contract
+ Resource
+ Operation
+ Classification
+ Destination
+ Version
+ Approval
+ Policy Version
→ allow / deny / require_approval
```

競合時は最も制限の強い判定を採用し、高RiskでPolicy評価に失敗した場合はFail Closedします。

## 5.4 Data Flow

Tool Allowlistだけでは、Read → Model → External Writeの流出を防げません。

- InputへClassification
- Derived OutputへLabel伝播
- Sinkごとの上限
- Tenant / Purpose / Destination
- Independent DLP
- Redaction Evidence
- 高機密Readと外部Writeの能力分離

Modelの要約だけで分類を下げません。

## 5.5 Human Gate

Trusted UIは、正規化引数、対象Version、Diff、Destination、Amount、Irreversibility、Action Hash、Policy、Rollbackを信頼できるSourceから表示します。

承認後、実行直前にHash、Version、Expiry、Policyを再確認します。

---

# 6. State、Checkpoint、Failure Recovery

## 6.1 保存State

- task_id / contract_hash
- current_phase
- completed / pending criteria
- decisions
- changed_resources
- evidence_refs
- policy / approvals
- budget_usage
- retry_counts
- uncertain_actions
- last_checkpoint
- stop_reason

## 6.2 Checkpoint

- Current Goal
- Active Contract
- Current State
- Last committed step
- Pending action
- External Request IDs
- Idempotency Keys
- Expected Versions
- Active Lease
- Valid Approvals
- Reconciliation action

## 6.3 Error分類

- transient
- invalid_input
- permission
- policy
- conflict
- permanent
- uncertain
- unknown

Retryはtransient等、再実行しても副作用が安全な場合だけ行います。

## 6.4 Graceful Degradation

- Write停止、Readのみ継続
- Search停止時はSource付き回答を停止
- Approval基盤停止時は高Risk Actionを拒否
- Memory停止時はCurrent Sessionのみ
- 大型Model停止時は低Risk要約だけ
- Policy停止時は高Risk Fail Closed

---

# 7. Verification、Evidence、Completion

## 7.1 Verificationの順序

```text
Schema
→ Type
→ Static Analysis
→ Unit
→ Integration
→ Contract / Business Rule
→ E2E
→ LLM Review
→ Human Review
```

決定論的に判定できるものを先に使います。

## 7.2 Criterion-level Result

[Verification Result Schema](../core/schemas/verification-result.schema.json)を使い、Criterion、Verifier、Subject Version、Evidenceを結び付けます。

Status:

- pass
- fail
- not_run
- blocked
- waived

## 7.3 Evidence Quality

- Relevance
- Completeness
- Independence
- Tamper Resistance
- Freshness
- Subject Binding
- Reproducibility
- Uncertainty

Riskに応じてAgent Evidence、Deterministic Evidence、Trusted CI、Independent Reviewer、Human Observationを組み合わせます。

## 7.4 Completion

[Completion Decision Schema](../core/schemas/completion-decision.schema.json)へ、Criteria、Policy、Approval、Scope、Unresolved Error、Uncertain Action、Residual Riskを集約します。

Agentの終了とTask Acceptanceを分けます。

---

# 8. EvaluationとReview

## 8.1 4層

- Component
- Trajectory
- Outcome
- Operational

## 8.2 Dataset

- Version
- Development / Holdout / Incident / Adversarial
- Starting State
- Task Contract
- Expected Criteria
- Allowed / Forbidden Scope
- Required Evidence
- Risk Band
- Reviewer Rubric
- Environment / Tool Version

## 8.3 Release Decision

- Sample数
- 信頼区間
- Task / Risk /規模の層別
- Paired Comparison
- Flaky Policy
- Contamination
- Severity-weighted Failure
- Safety上側信頼限界
- Production Drift
- Cost per Accepted Task

Safety Failure 0件だけで安全と判断しません。

## 8.4 Reviewer Assurance

Context、Workspace、Permission、Evaluator、Execution、Expertiseの独立性を別々に記録します。

---

# 9. ObservabilityとReplay

## 9.1 Trace

```text
session
└─ task
   └─ turn
      └─ step
         ├─ model_call
         ├─ retrieval
         ├─ memory
         ├─ policy
         ├─ tool_call
         ├─ verification
         ├─ approval
         └─ state_transition
```

最低限:

- trace / session / task / step IDs
- Contract / Policy / Harness Version
- Model
- ToolとRedact済み引数
- Result Status
- Latency / Token / Cost
- Retry
- State before / after
- Evidence
- Error / Stop Reason
- Action Hash
- External Request ID

## 9.2 Privacy

Prompt、Source、PII、Token、Credentialを無制限に保存しません。Masking、Retention、Access、Tenant Isolation、Export Auditを設計します。

## 9.3 Replay

- Audit Replay: EventからState再構築
- Simulation Replay: Tool Result固定でHarness比較
- Live Re-execution: 新しいRunとして外部Toolを再実行

Live Re-executionは新しい認可、Approval、Idempotencyを要求します。

---

# 10. Securityと人間参加

## 10.1 Untrusted Input

User、Web、Email、Attachment、RAG、Tool Result、Subagent Output、Memoryを非信頼入力として扱います。

目標は「Modelが絶対にInjectionへ騙されない」ではありません。

```text
悪意ある指示を読む
→ 危険Actionを提案する可能性
→ Policy / Permission / PEPが拒否
→ 事故へ進まない
```

## 10.2 Threat Model

- Data Exfiltration
- SSRF
- Cross-tenant Leakage
- Malicious Tool / MCP / Skill
- Supply Chain
- Memory / RAG Poisoning
- Log / Trace Leakage
- Approval Substitution
- Indirect Prompt Injection
- Privilege Escalation
- Sandbox Escape
- Agent-to-Agent Propagation
- Denial of Wallet / Cost Exhaustion
- Stale Policy / Approval Replay

## 10.3 Least Privilege

- Read / Write分離
- Path / Resource Scope
- Tenant Bound
- Short-lived Credential
- Network Default Deny
- Sink Allowlist
- Permission TTL
- Revocation
- Dedicated Production Path

---

# 11. Multi-Agent

## 11.1 正当な理由

- Context分離
- Parallel Read
- Permission分離
- Model / Tool分離
- Independent Review
- Long-running Unit

役割名を増やすだけでは品質は上がりません。

## 11.2 Handoff Contract

- Goal
- Criteria
- Scope
- Input
- Allowed Tool
- Forbidden Action
- Output Schema
- Evidence
- Budget
- Stop Condition
- Base Version

Raw Transcript全体を渡しません。

## 11.3 Parallel Write

- Worktree / Resource分離
- Ownership
- Public Contract固定
- Shared Generated File回避
- Merge順
- 各Branch Verification
- 統合後Verification
- Conflictを黙ってModel任せにしない

---

# 12. High-risk Operation

[本番高リスクプロファイル](../profiles/production-high-risk.md)を適用します。

必須候補:

- Independent Enforcement
- Multi-axis Risk
- Data Flow / DLP
- Short-lived Credential
- Trusted Approval UI
- Action Hash
- Separation of Duties
- Trusted Evidence
- Immutable Audit
- Idempotency / Reconciliation
- Emergency Stop
- Compensation
- Incident Runbook

---

# 13. Distributed Execution

[分散実行プロファイル](../profiles/distributed-execution.md)を適用します。

## 13.1 At-least-once

重複配信を前提に、Idempotency KeyとResult照会を持ちます。「Exactly once」はTransaction境界を説明せずに使いません。

## 13.2 LeaseとFencing Token

Leaseだけでは、失効した古いWorkerが実行を続ける可能性があります。Lease取得ごとに単調増加するFencing Tokenを発行し、State StoreやTool Gatewayが古いTokenを拒否します。

外部ProviderがTokenを検証できない場合、限界とReconciliationを明記します。

## 13.3 CAS

期待Versionが一致する場合だけ更新します。失敗時は再読込みしてReplanします。

## 13.4 Outbox / Inbox

Local DBと外部Systemを一つのTransactionにできない場合、Request ID、Idempotency、Outbox / Inbox、Webhook、Status API、Reconciliation、Compensationを組み合わせます。

---

# 14. 導入と成熟度

## Stage 0: Generation

- Single Model Call
- 外部副作用なし
- 人間が全出力を確認

## Stage 1: Tool Use

- 少数Tool
- Schema
- Budget
- Structured Error
- Basic Log

## Stage 2: Production Single Agent

- State
- Idempotency
- Verification
- Evaluation
- Permission
- Observability
- Cost

## Stage 3: High-risk / Long-running

- Approval
- Checkpoint
- Audit
- Sandbox
- Reconciliation
- Degradation

## Stage 4: Multi-Agent / Distributed

- Handoff
- Parent-child Trace
- Spawn Budget
- Independent Permission
- Fencing
- Outbox / Inbox

順序:

```text
Completion Criteria
→ Tool Validation
→ Budget / Stop
→ Event
→ Eval Case
→ Permission / Idempotency
→ State / Resume
→ Cost / Performance
→ Memory
→ Multi-Agent / Distribution
```

---

# 15. 実装標準への適合

最低限、次のArtifactを持ちます。

- [Task Contract](../core/schemas/task-contract.schema.json)
- [Policy Decision](../core/schemas/policy-decision.schema.json)
- [Approval Record](../core/schemas/approval-record.schema.json)
- [Verification Result](../core/schemas/verification-result.schema.json)
- [Completion Decision](../core/schemas/completion-decision.schema.json)

Schemaは形を保証します。参照先の真正性、Hash計算、Policy意味、Evidenceの妥当性はRuntimeとTrusted Environmentで検証します。

## 参考入口

- [中核設計原則](../core/principles.md)
- [共通用語集](../core/glossary.md)
- [統合判定モデル](../core/decision-model.md)
- [エージェントランタイム](../learning/intermediate/agent-runtime.md)
- [証拠とポリシー](../learning/intermediate/evidence-and-policy.md)
- [CI・レビュー・評価](../learning/intermediate/ci-review-and-evaluation.md)
