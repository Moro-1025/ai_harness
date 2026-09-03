---
title: "Profile: チーム・Pull Request・Trusted CI"
version: "1.0"
core_version: "1.0"
profile_id: "team-ci"
risk_scope: "medium-to-high"
updated: "2026-09-02"
---

# Profile: チーム・Pull Request・Trusted CI

## 目的

AIエージェントの変更を、個人Sessionの結果ではなく、固定Commit、共通Policy、独立Evidence、Reviewによってチームが採用できるようにします。

---

## 適用範囲

- Team Repository
- Pull Request
- CI Review
- CI Patch Generation
- Scheduled Repository Analysis
- Documentation更新
- Medium-risk Code Change
- High-risk変更の事前検証

Production Release自体は[本番高リスクプロファイル](production-high-risk.md)へ分離します。

---

## Architecture

```text
Task Owner
  ↓
Task Contract + Contract Hash
  ↓
Writer Agent / Branch
  ↓
Patch or Commit
  ↓
Trusted Deterministic CI
  ↓
Fresh read-only Reviewer
  ↓
Human Code Owner
  ↓
Completion Decision
```

---

## Role

| Role | Permission | 責任 |
|---|---|---|
| Task Owner | 契約作成 | 目的、Scope、Criteria、Risk |
| Writer | workspace-write | Patchと自己検証 |
| CI Verifier | read-only + test execute | Fixed Commitの独立検証 |
| Agent Reviewer | read-only | Finding |
| Code Owner | Merge判断 | Requirement、Architecture |
| Security | 高Risk Review | Policy、Threat、Risk |
| Harness Owner | Config変更 | Profile、Script、Policy、Eval |

Writerと最終Acceptance Ownerを同一の自動主体にしません。

---

## Capability

### Writer

```yaml
filesystem: workspace-write
network:
  mode: denied
credential_access: none
external_side_effects: denied
git:
  commit: allowed-on-task-branch
  push: task-specific
  merge: denied
  release: denied
```

### Trusted Verifier

```yaml
filesystem: read
execution:
  tests: allowed
network:
  mode: allowlisted
credential_access: verification-only
external_side_effects: denied
subject:
  fixed_commit: required
artifact_store:
  writer_mutable: false
```

### Reviewer

```yaml
filesystem: read
workspace: clean-checkout
context: fresh
network: denied-or-documentation-only
credential_access: none
write_findings_only: true
```

---

## Repository要件

```text
AGENTS.md
Task Contract Template
scripts/agent/bootstrap
scripts/agent/doctor
scripts/agent/verify
JSON Schemas
Policy
Branch Protection
Required CI
Artifact Retention
Incident Runbook
Eval Dataset
```

Harness Fileの変更にはCode OwnerまたはHarness OwnerのReviewを要求します。

例:

- Agent instruction
- Tool / MCP Config
- Hook
- Rule
- Permission Profile
- CI Workflow
- Verification Script
- Eval Rubric

---

## Pull Request要件

PRへ含めるもの:

- Task / Issue
- Contract ID / Hash
- Starting Commit
- Scope
- Completion Criteria
- Criterion-level Verification
- Changed Files
- Not Run
- Residual Risk
- Rollback
- Agent Product / Version
- Harness Version

PR説明はEvidenceへの索引です。

---

## Trusted CI

### 必須原則

- Clean Checkout
- Fixed Commit
- Reproducible Environment
- Trusted Script
- Correct Exit Code
- Result Schema
- Artifact Hash
- WriterがResultを書き換えられない
- Secret Masking
- Retention / Access
- Required Check

### Pipeline例

```text
lint
type
unit
integration
contract
secret-scan
scope-check
evidence-binding
agent-review
completion-decision
```

High-risk CriterionはAgent ReviewだけでPassにしません。

---

## Policy

最低限:

```text
deny > require_approval > allow
```

Policy Decisionへ保存:

- Subject
- Contract Hash
- Resource
- Operation
- Data Classification
- Destination
- Target Commit
- Approval
- Policy Version
- Input Hash
- Reason
- Obligation

Policy Engine障害時、高Risk変更はFail Closedします。

---

## Reviewer Assurance

Medium Risk:

```yaml
context: fresh
workspace: clean_checkout
permissions: read_only
evaluator: same-or-different-model
execution: trusted_ci
human_review: required
```

High Risk:

```yaml
context: fresh
workspace: clean_checkout
permissions: read_only
evaluator: different-model-or-method
execution: trusted_ci
expertise:
  - domain
  - security
human_review: required
```

---

## Evaluation

Harness変更前後で同じDataset Versionを実行します。

最低限報告:

- Case数
- Task Type / Risk層別
- Task Success
- Scope Violation
- Verification Skip
- First-pass Acceptance
- Rework
- Human Review Minutes
- Cost per Accepted Change
- Policy Violation
- Critical / High Failure
- Flaky
- Contamination Review
- Drift

Critical Failureを平均Scoreで相殺しません。

---

## CredentialとNetwork

- Long-lived SecretをRepository-controlled Codeと同じJob全体へ置かない
- Short-lived Credential
- Fork / Untrusted Branchから到達させない
- Agent ProcessだけへScope
- Network Allowlist
- Package Lifecycle Scriptに注意
- ArtifactへCredentialを含めない
- Run終了後に破棄
- Egress Log

---

## Stop Conditions

- Required CIが利用不能
- Fixed Commitを検証できない
- Contract Hash不一致
- Harness / Policyが無審査で変更
- Secret Exposure
- Scope Violation
- Critical / High Finding未解決
- Required Evidence不足
- Approval期限切れ
- Uncertain Actionが残る
- Holdout / Incident Setで回帰
- Artifact Storeへ保存不能

---

## Profile適合条件

- Task ContractとHashをPRへ結び付ける
- Writer、Verifier、Reviewer、Human Ownerを分ける
- Fixed CommitをTrusted CIで検証する
- Branch ProtectionとRequired Checkがある
- CriterionごとのVerification Resultがある
- Agent Reviewを唯一のGateにしない
- Harness変更自体をReview・Evalする
- CredentialとRepository-controlled Codeを分離する
- ArtifactのRetention、Access、Tamper Resistanceがある
- IncidentをEval Caseへ追加する
- Sample数・層別・不確実性を含むRelease Gateがある

## 関連文書

- [チームハーネス](../learning/intermediate/team-harness.md)
- [CI・レビュー・評価](../learning/intermediate/ci-review-and-evaluation.md)
- [証拠とポリシー](../learning/intermediate/evidence-and-policy.md)
- [Codexプロファイル](codex.md)
