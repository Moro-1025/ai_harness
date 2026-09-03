---
title: "AIコーディングエージェント活用リファレンス"
subtitle: "Repository、Git、CI、権限、検証を含む上級参照文書"
version: "1.2"
core_version: "1.0"
audience: "advanced-reference"
updated: "2026-09-02"
---

# AIコーディングエージェント活用リファレンス

> Codex等のAIコーディングエージェントを、単なるコード生成Toolではなく、**権限・コンテキスト・変更範囲・検証・証拠・採用判断を管理された開発プロセス**として運用するための参照文書です。

共通原則と用語は`docs/core/`を正本とします。本書はProject Harnessへの適用を扱い、製品固有の機能名・設定は[Profiles](../profiles/)へ分離します。

旧v1.1全文は[ルートの旧版](../../AI_Coding_Agent_Usage_Harness_Handbook_JA.md)に残しています。

---

# 1. Vendor HarnessとProject Harness

```text
人間・Team・CI
  ↓
Project Harness
  ├─ Task Contract
  ├─ Repository Guidance
  ├─ Skill / Script / Policy
  ├─ Permission / Git Isolation
  ├─ Verification / Evidence
  └─ Acceptance / Eval
  ↓
Vendor Harness
  ├─ Agent Loop
  ├─ Tool Execution
  ├─ Session / Context
  └─ Approval UI
  ↓
Model
```

Vendorが提供する機能だけでは、次は決まりません。

- どこを変更してよいか
- どのTestが必須か
- 何を完了とするか
- どのCommandが危険か
- どこへNetwork接続してよいか
- 何を証拠として採用するか
- 誰がMerge・Releaseを決めるか

Project Harnessは、Vendor能力を自分たちの開発規約へ適合させる層です。

---

# 2. 7つの面

| 面 | 主な責任 | 実装例 |
|---|---|---|
| 指示 | 安定したProject規則 | `AGENTS.md`、参照文書 |
| Task | 目的、Scope、Criteria | Task Contract、Plan |
| 能力 | Tool、外部Data、手順 | Built-in Tool、MCP、Skill、Script |
| 権限 | File、Process、Network、Secret | Sandbox、Policy、Approval |
| 隔離 | 変更とEnvironment | Branch、Worktree、Container |
| 検証 | 正しさ、安全、Evidence | Test、Lint、Type、Review、CI |
| 運用 | Trace、Eval、Cost、Incident | JSONL、OTel、Artifact、Scorecard |

一つのMechanismへ複数責任を詰め込みません。

---

# 3. Repositoryと開始状態

## 3.1 Bootstrap

人間とAgentが同じCommandを使えるようにします。

```text
scripts/agent/bootstrap
scripts/agent/doctor
scripts/agent/verify
scripts/agent/smoke
```

良いScript:

- 非対話
- 正しいExit Code
- 失敗理由が短い
- Log場所が一定
- Environmentを暗黙に破壊しない
- Secretを表示しない
- Machine-readable Summary

## 3.2 Baseline

Task開始時:

```text
Git status
Current Branch
Starting Commit
Runtime Version
Dependency
Build
Targeted Test
Required Service
Permission
Network
```

Baseline FailureとAgentが作ったRegressionを分けます。

## 3.3 Project Trust

Repository内のConfig、Hook、Rule、Skill Script、Package Lifecycle、Container、CI Workflowは実行能力を持ちます。

初めて開くRepositoryではSourceだけでなくHarness定義をReviewします。Trustは「Codeが正しい」という意味ではなく、Repositoryが提示するAgent設定を読み込んでよいという判断です。

---

# 4. Task Contract

良い依頼は「何をするか」より「何を成立させるか」を示します。

```text
目的
期待状態
Scope
Forbidden Scope
Non-goal
Constraint
Completion Criteria
Evidence
Risk / Capability
Human Gate
Stop Condition
```

Task PromptへRepository全体を貼りません。Issue、再現、Entry Point、関係Document等、探索の入口を示します。

## 4.1 Task Size

分割原則:

```text
独立Verification
+ 独立Review
+ 独立Rollback
```

大きすぎるTaskを一つの長いSessionへ詰め込まないようにします。

## 4.2 Contract Amendment

Public API、Migration、Dependency、Security境界等が調査中に必要と分かった場合、Agentが暗黙にScopeを広げません。

- Stop
- Amendment
- Risk再評価
- Reapproval
- 新Contract Hash
- 新Branchまたは明確な継続
- Evidence再評価

---

# 5. 永続指示とTask固有情報

```text
常時必要な短い規則
→ Repository Guidance

Task固有の目的・Scope
→ Task Contract

特定作業の判断手順
→ Skill

決定論的処理
→ Script

Lifecycleで必ず実行
→ Hook

Command許可・禁止
→ Rule

外部Data / Action
→ MCP / Tool

最終境界
→ OS / Network / Credential / CI / Git Protection
```

永続指示を肥大化させると、Context消費、指示競合、古い規則、重要事項の埋没が増えます。

同じ内容をPrompt、Guidance、Skillへ重複させず、正本とTriggerを決めます。

---

# 6. Plan、Execute、Verify

## 6.1 Planが必要なTask

- 原因不明
- 複数Subsystem
- Public API
- Migration
- Security境界
- Dependency追加
- UI / Backend同時
- 複数Environment
- 長時間

小さく明確な変更へ重いPlanを強制しません。

## 6.2 PermissionをPhaseで分ける

```text
Plan
  read-only
  必要時だけSearch / Read-only MCP

Execute
  workspace-write
  Network原則off
  Scope内

Verify
  read-onlyまたは限定実行
  Fixed Subject

Release
  Coding Sessionと分離
  Dedicated Environment
  Human Gate
```

## 6.3 Step Verification

小さなChangeごとにTargeted Testを実行し、最後にIntegration / Full / Diff Reviewへ広げます。

---

# 7. Permission、Network、Secret

## 7.1 2軸

```text
技術的実行境界
= File / Process / Network / Credential

承認境界
= いつ人間・Reviewerへ送るか
```

承認Promptがあっても、Sandboxが広すぎれば誤承認時の損失が大きくなります。

## 7.2 Profile

| Task | File | Network | Approval |
|---|---|---|---|
| 調査 | read-only | off | 原則不要 |
| Code Review | read-only |限定 | 外部Action時 |
| 実装 | workspace-write | off | Scope越え |
| Dependency | workspace-write | Registry Allowlist | Install前 |
| CI Review | read-only |最小 | Fail Closed |
| CI Patch | Isolated workspace |最小 | 事前Policy |
| Release | Dedicated |Allowlist | Human必須 |

## 7.3 Network

Networkは便利機能ではなく外部入出力の権限です。

- Default off
- Task単位
- Domain / Destination Allowlist
- External ContentはUntrusted
- DLP / Egress
- Audit

Web Search、Command Network、MCPを同じ能力として扱いません。

## 7.4 Secret

- Promptへ貼らない
- `.env`全体を読ませない
- Repository-controlled Codeと長期Keyを同居させない
- Short-lived Credential
- Secret Broker
- Process Scope
- Audit
- Revocation
- Artifact / Transcriptへ残さない

---

# 8. Skill、Script、Hook、Rule、MCP、Subagent

| Mechanism | 主な用途 |
|---|---|
| Guidance | 常時の短い規則 |
| Skill | 判断を含む再利用Workflow |
| Script | 同じ入力を決定論的に処理 |
| Hook | Lifecycleで自動実行 |
| Rule | Command Prefix等のPolicy |
| MCP / Tool | 外部Data・Action |
| Subagent | Context、権限、Model、役割の分離 |
| Profile | Task種別の設定Preset |

## 8.1 Skill

- 一つのJob
- Trigger / Non-trigger
- Input / Output
- Evidence
- Stop Condition
- Scriptは必要な場合だけ

## 8.2 Hook

- Secret検知
- Pre-tool Policy
- Tool後のLog
- Stop時Verification
- Compaction前Checkpoint
- Subagent Result Validation
- Evidence Bundle生成

Hook間の暗黙順序へ依存せず、Idempotent、Timeout、Failure Policyを持たせます。

## 8.3 Rule

単純でReview可能なCommand Policyへ使います。Shell全体の意味、File内容、Business RuleをRuleだけで完全に理解できると考えません。

## 8.4 MCP / External Tool

- Tool一覧
- Read / Write / Destructive
- Credential Scope
- Server Instruction
- Data Destination
- Timeout / Retry
- Required / Optional
- Audit
- Tenant
- OAuth Scope

MCPというProtocol自体を安全境界とみなしません。

---

# 9. Git、Branch、Worktree

## 9.1 安全装置

開始:

- `git status`
- Starting Commit
- Existing Diff
- Untracked File

終了:

- File List
- Diff
- Test
- Generated Artifact
- Untracked File
- Patch Hash

## 9.2 基本単位

```text
1 Task
1 Branch / Worktree
1 Contract Hash
1 Completion Report
1 Evidence Bundle
```

## 9.3 Worktree

向く場面:

- 人間が別作業中
- Parallel Task
- Long-running
- Experiment
- Clean Review

注意:

- Ignored File不足
- Dependency
- Port / DB競合
- Cache汚染
- Secret複製
- Absolute Path

WorktreeはFileとGit Stateの隔離であり、Context・Credential・Networkも別途分離します。

## 9.4 Commit / Push / Merge / Release

同じ権限として扱いません。

```text
Edit
< Commit
< Push
< Merge
< Release
< Production Change
```

上位ほどIndependent ControlとHuman Gateを強くします。

---

# 10. Context、Session、Artifact

Main Sessionへ残す:

- Goal
- Contract
- Plan
- Decision
- Current State
- Human Feedback
- Evidence参照

退避:

- 大量Log
- Search全文
- Raw Subagent Transcript
- Full Source
- Binary

Session Artifact例:

```text
.agent-runs/<run-id>/
├─ task.json
├─ plan.md
├─ checkpoint.json
├─ config-summary.json
├─ git-before.txt
├─ events.jsonl
├─ verification/
├─ diff.patch
├─ review.json
└─ completion.json
```

Retention、Secret、PII、Access、Tamper Resistance、Source of Truthを定義します。

新Sessionへ移る基準:

- Task変更
- Scope大幅変更
- Plan全面変更
- Model / Permission変更
- Context Pollution
- Independent Review
- 別Experiment

---

# 11. Verification、Evidence、Acceptance

順序例:

```text
Static
→ Targeted Test
→ Integration
→ Smoke / E2E
→ Diff Review
→ Independent Review
→ Human Acceptance
```

すべてのTaskで全段階は不要です。CriterionとRiskに応じて選びます。

## 11.1 verify Script

Project側で入口を固定します。

```text
scripts/agent/verify --scope auth
scripts/agent/verify --changed
scripts/agent/verify --full
```

## 11.2 Evidence

- Agent実行
- Project Script
- Trusted CI
- Independent Reviewer
- Human Observation

Criterionごとに[Verification Result](../core/schemas/verification-result.schema.json)を作ります。

## 11.3 Acceptance Gate

機械判定:

- Required Test
- Lint / Type
- Scope
- Forbidden File
- Secret
- Contract / Subject Binding
- Policy / Approval
- Completion Decision

人間判断:

- Requirement
- Architecture
- UX
- Trade-off
- Security Judgment
- Future Maintenance

---

# 12. SubagentとParallel Work

正当な理由:

- Context分離
- Read並列
- Independent Review
- Tool / Permission分離
- Model分離
- Long-running Unit

最初はExplorer、Test Triage、Documentation、Security Review、Log分析等のRead-heavy Taskから始めます。

Handoff:

- Narrow Goal
- Scope
- Input
- Allowed Tool
- Forbidden Action
- Output Schema
- Evidence
- Stop Condition
- Base Commit

Parallel Write条件:

- Separate Worktree
- Non-overlapping Ownership
- Contract固定
- Shared Generated File回避
- Integration Owner
- Each Branch Verification
- Post-merge Verification

---

# 13. CIと非対話実行

開始前に固定:

- Input
- Permission
- Timeout
- Schema
- Exit
- Required Integration
- Evidence
- Failure Policy

構成:

```text
Deterministic CI
+
Agent Review
+
Human Review
```

Agent Reviewだけを唯一のGateにしません。

保存:

- Product / CLI Version
- Model
- Profile
- Starting Commit
- Task Contract
- Enabled Tool
- Permission
- Event
- Output
- Patch
- Verification
- Policy / Approval

詳細は[CI・レビュー・評価](../learning/intermediate/ci-review-and-evaluation.md)と[チームCIプロファイル](../profiles/team-ci.md)を参照してください。

---

# 14. MetricsとEvaluation

見る指標:

- Task Success
- First-pass Acceptance
- Verification Pass
- Scope Violation
- Tool Failure
- Rework
- Human Review Minutes
- Time to Accepted Change
- Cost per Accepted Change
- Policy Violation
- Rollback
- Incident

Harness変更前後で同じVersion付きDatasetを実行します。Task Type、Risk、Repository規模で層別し、Sample数、信頼区間、Flaky、Contamination、Driftを報告します。

「生成行数」「起動Agent数」を成果指標にしません。

---

# 15. Incidentと継続改善

```text
Stop
→ Side Effect確認
→ Branch / Worktree隔離
→ Credential失効
→ Evidence保全
→ Rollback / Compensation
→ Root Cause
→ Harness Control修正
→ Eval Case
```

改善先:

| Cause | Target |
|---|---|
| Task曖昧 | Contract |
| Stable Rule不足 | Guidance |
| Workflow不足 | Skill |
| Non-deterministic手順 | Script |
| Lifecycle Control不足 | Hook |
| Command Policy不足 | Rule |
| Excessive Tool | MCP整理 |
| Scope Conflict | Worktree / Ownership |
| Verification不足 | Verify / Acceptance |
| Permission過大 | Sandbox / Credential |
| Eval不足 | Dataset |

定期的に古いRule、Hook、Prompt、Toolを削除します。

---

# 16. 成熟度

## Stage 0 Prompt Only

人間が毎回指示し目視。

## Stage 1 Repository Guidance

Task Template、Guidance、Build / Test、Branch。

## Stage 2 Verification Harness

Verify Script、Completion Report、Evidence、Permission Profile、Diff Review。

## Stage 3 Policy Harness

Rule、Hook、Network、Secret、Worktree、Fresh Review。

## Stage 4 Reusable Workflow

Skill、MCP、Subagent、Handoff、Task Profile。

## Stage 5 Automation and Evaluation

Non-interactive、Structured Event、CI、Telemetry、Eval、Harness Release Gate。

順序:

```text
Task Contract
→ Repository Guidance
→ Git Isolation
→ Verification
→ Completion Report
→ Permission
→ Policy / Hook
→ Skill
→ External Tool
→ Subagent
→ CI
→ Eval / Telemetry
```

MCPやMulti-Agentから始めません。

---

# 17. Product Profile

製品名ではなく、次のCapabilityを確認します。

- Persistent Instruction
- Task Instruction
- Read-only / Write Scope
- Network
- Credential
- Command Policy
- Lifecycle Hook
- External Tool
- Session / Resume
- Structured Event
- Non-interactive
- Subagent Permission
- Worktree / Branch
- Output Schema
- Config Version

Codexへの対応は[Codexプロファイル](../profiles/codex.md)に分離しています。

## 参考入口

- [中核設計原則](../core/principles.md)
- [統合判定モデル](../core/decision-model.md)
- [コーディングエージェント初級チュートリアル](../learning/beginner/coding-agent-tutorial.md)
- [チームハーネス](../learning/intermediate/team-harness.md)
- [個人ローカルプロファイル](../profiles/personal-local.md)
- [チームCIプロファイル](../profiles/team-ci.md)
