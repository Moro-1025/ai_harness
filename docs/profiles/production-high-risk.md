---
title: "Profile: 本番・高リスクAIエージェント"
version: "1.0"
core_version: "1.0"
profile_id: "production-high-risk"
risk_scope: "high-to-critical"
updated: "2026-09-02"
---

# Profile: 本番・高リスクAIエージェント

## 目的

Production、Restricted Data、外部副作用、金銭、権限、法務、不可逆操作を扱う場合の追加Controlを定義します。

このProfileは安全性を保証する認証ではありません。対象業務のThreat Model、法務、規制、契約、Risk Appetite、運用体制に合わせた個別設計が必要です。

---

## 適用Trigger

次のいずれか:

- Production Resource
- 顧客・従業員・患者等の機密Data
- Credential、Token、Key
- Cross-tenant
- 外部メール・公開・Issue作成
- 決済・返金・送金
- 大量変更・削除
- 権限付与・失効
- 契約・法的効果
- 人間が毎回Outputを確認しない
- Rollback困難
- 誤りの検出が難しい
- 復旧時間が長い

---

## 必須Architecture

```text
Untrusted Input
  ↓
Model / Agent（提案）
  ↓
Tool Gateway
  ├─ Schema
  ├─ Identity
  ├─ Policy Decision
  ├─ Data Flow / DLP
  ├─ Approval
  ├─ Idempotency
  └─ Audit
  ↓
Independent Enforcement
  ├─ OS / Container
  ├─ Network / Egress
  ├─ Credential Broker
  ├─ Database Authorization
  └─ Provider-side Control
  ↓
Business System
```

Modelと同じWorkspace・権限で変更できるHookだけを最終境界にしません。

---

## Risk Assessment

必須軸:

- Confidentiality
- Integrity
- Availability
- External Side Effect
- Reversibility
- Financial / Legal
- Credential Access
- Tenant Boundary
- Detection Difficulty
- Recovery Time

各軸へ次を記録:

- Asset
- Threat Actor
- Failure Mode
- Impact
- Likelihood
- Existing Control
- Residual Risk
- Owner
- Acceptance

Risk Bandだけから権限を決めません。

---

## Capability

Default Deny:

```yaml
filesystem: none
network: denied
external_side_effects: denied
credential_access: none
allowed_tools: []
```

Task ContractとPolicyにより最小能力だけを一時付与します。

### Credential

- Short-lived
- Audience / Resource scoped
- Tenant bound
- Operation scoped
- Agent Contextへ平文で渡さない
- Dedicated Tool / Broker
- Automatic expiry
- Revocation
- Usage audit
- No credential reuse across tasks

### Network

- Egress Proxy
- DNS / IP検証
- Domain / Endpoint Allowlist
- Metadata Service遮断
- Private Network境界
- Request / Response Size上限
- Protocol制限
- DLP
- Destination audit

---

## Data Flow

Label:

```text
public < internal < confidential < restricted
```

実際の分類体系へ読み替えます。

必須:

- Input Classification
- Derived Output伝播
- Tenant
- Purpose
- Destination
- Sink Maximum Classification
- Model-independent DLP
- Redaction Evidence
- Output Validation
- Log / TraceもSinkとして評価
- Cross-tenant Cache / Memory分離

Modelの要約・言い換えだけでClassificationを下げません。

---

## Policy Decision

Input:

```text
Subject
+ Contract Hash
+ Resource / Tenant
+ Operation
+ Classification
+ Destination
+ Expected Version
+ Action Hash
+ Approval
+ Policy Version
```

Output:

- allow
- deny
- require_approval
- reasons
- obligations
- expiry

高RiskでPolicy評価に失敗した場合はDenyします。

Policy変更はVersion管理し、独立Review、Test、Rollout、Rollbackを持たせます。

---

## Human Gate

Trusted Approval UIへ表示:

- Operation
- Contract Hash
- Normalized Arguments
- Target Resource / Version
- Actual Diff
- Destination
- Amount / Count
- Irreversibility
- Action Hash
- Policy Decision
- DLP Result
- Rollback / Compensation
- Residual Risk
- Approval TTL

### Separation of Duties

Riskに応じて:

- Task CreatorとApproverを分離
- WriterとApproverを分離
- Domain Approver
- Security Approver
- Two-person Approval
- Amount / Volume threshold
- Approval expiry
- One-time consumption

### Break-glass

- Normal Flowでは実行不能
- Strong Authentication
- Narrow Capability
- Short TTL
- Reason必須
- Incident ID
- Real-time alert
- Immutable Audit
- Mandatory post-review
- Automatic revoke

---

## External Side Effect

必須:

- Idempotency Key
- Payload Hash
- Expected Version
- External Request ID
- Provider Result照会
- `uncertain` State
- Reconciliation
- Duplicate Detection
- Compensation
- Stop / Kill Switch

TimeoutをFailureとして無条件再送しません。

---

## VerificationとEvidence

High-risk Criterion:

- Deterministic Verification
- Fixed Subject
- Independent Process
- Trusted CIまたはTrusted Runtime
- Tamper-resistant Artifact
- Policy / Approval Binding
- Human Observation where needed
- Explicit Not Run / Limitation

EvidenceをWriter Workspaceだけへ置きません。

Completion前:

```text
Required Criteria pass
+ Evidence current
+ Scope pass
+ Policy pass
+ Approval valid
+ DLP pass
+ No critical unresolved errors
+ No uncertain actions
+ Residual Risk accepted
```

---

## ObservabilityとAudit

記録:

- Identity
- Contract / Policy / Harness Version
- Action proposal
- Normalized Arguments
- Policy Decision
- Approval
- Action Hash
- Tool Result
- External Request ID
- Side Effect
- Verification
- Completion Decision
- Stop Reason
- Cost
- Tenant / Purpose
- Redaction status

保護:

- Append-only
- Access Control
- Tenant Isolation
- Retention
- Encryption
- Hash / Signature / Object Lock as needed
- Export Audit
- PII最小化
- Secret Masking
- Deletion / Legal Hold方針

---

## Security Testing

- Prompt Injection
- Indirect Injection
- Data Exfiltration
- SSRF
- Cross-tenant
- Malicious Tool / MCP
- Supply Chain
- Memory Poisoning
- Approval Substitution
- Stale Approval Replay
- Privilege Escalation
- Sandbox Escape
- Denial of Wallet
- Duplicate Side Effect
- Timeout / Uncertain
- Policy Engine Failure
- Logging Leakage

Critical FailureはRelease Blockです。

---

## Operational Control

- Emergency Stop
- Write Disable
- Network Disable
- Credential Revoke
- Tenant Quarantine
- Rate / Amount / Count limit
- Circuit Breaker
- Feature Degradation
- Manual fallback
- On-call owner
- Incident Runbook
- Recovery objective
- Reconciliation dashboard

---

## Rollout

```text
offline eval
→ simulation replay
→ shadow read-only
→ internal limited
→ low-volume approved action
→ staged tenant / percentage
→ monitored production
```

各段階にExit CriteriaとRollbackを設定します。

Production FailureをEval Caseへ戻します。

---

## Stop Conditions

- Policy / DLP / Approval Service unavailable
- Contract / Action Hash mismatch
- Target Version mismatch
- Approval expired / revoked / consumed
- Credential Scope mismatch
- Tenant mismatch
- Destination not allowlisted
- Critical / High Finding
- Uncertain Side Effect
- Audit保存不能
- Budget / Count / Amount limit
- Kill Switch
- Drift threshold exceeded

---

## Profile適合条件

- Independent Enforcementがある
- Riskを多軸で評価する
- CapabilityはDefault Deny
- Short-lived CredentialとEgress Control
- Classification伝播とIndependent DLP
- Policy DecisionをVersion・Input Hash付きで保存
- Trusted Approval UIとAction Hash
- 必要なSeparation of Duties
- Idempotency、Uncertain、Reconciliation
- Trusted EvidenceとTamper-resistant Audit
- Emergency StopとCompensation
- Adversarial Evalと段階Rollout
- Critical Failureを平均で相殺しない
- 残存RiskのOwnerとAcceptanceがある

## 関連文書

- [統合判定モデル](../core/decision-model.md)
- [証拠とポリシー](../learning/intermediate/evidence-and-policy.md)
- [分散実行プロファイル](distributed-execution.md)
