---
title: "Product Profile: Codex"
version: "1.0"
core_version: "1.0"
profile_id: "codex"
product_docs_checked: "2026-09-02"
updated: "2026-09-02"
---

# Product Profile: Codex

## 目的

[AIコーディングエージェント活用リファレンス](../reference/coding-agent-usage-handbook.md)の責任を、Codexの現在の機能へ対応付けます。

このProfileはCodexの全機能を説明するものではありません。製品仕様は更新されるため、導入時は必ず公式Documentationと実際のCLI / App Versionを確認し、Config、Permission、Event SchemaをVersion固定してください。

> Codex固有の機能がない製品へ読み替える場合、同じ機能名を探すのではなく、同じ責任を満たせるかを確認します。

---

## 1. Capability Mapping

| Harness責任 | Codexでの代表的な面 |
|---|---|
| 永続Project指示 | `AGENTS.md`、`AGENTS.override.md` |
| 個人・Project設定 | Codex Config |
| File / Command Sandbox | `sandbox_mode`またはPermission Profile |
| 承認Policy | `approval_policy` |
| Workspace内Network | `sandbox_workspace_write.network_access` |
| Command Policy | Rules / `prefix_rule` |
| Lifecycle Control | Hooks |
| 再利用Workflow | Skills |
| 外部Data / Tool | MCP、Plugins / Apps |
| Context・権限分離 | Subagents / Custom Agents |
| 変更隔離 | Git Branch / Worktree |
| 非対話実行 | `codex exec` |
| Machine-readable Event | `codex exec --json` |
| Structured Final Output | `--output-schema` |
| Programmatic Integration | Codex SDK、App Server等 |

同じ責任をProject側のScript、CI、OS、Network、Git Protectionでも補完します。

---

## 2. AGENTS.md

Codexは作業開始前に`AGENTS.md`系の指示を探索し、GlobalからProject、Current Directoryへ向かって指示を重ねます。Current Directoryに近い指示が後から加わります。

推奨:

- RootにはRepository全体の短い規則
- Subdirectoryにはその領域固有の規則
- Task固有情報はTask Contractへ分離
- 詳細手順はSkillまたは参照Document
- Secretを書かない
- 機械的に強制できるものを長文で繰り返さない
- 有効な指示ChainをTask開始時に確認する

例:

```markdown
# AGENTS.md

- Task開始時に`task.md`を読む。
- 調査はread-onlyで開始する。
- 変更はTask Scopeと現在Worktree内だけ。
- Public API、DB Schema、Dependency変更前に停止する。
- 完了前に`scripts/agent/verify --changed`を実行する。
- Test未実行、変更File、残存Riskを報告する。
```

---

## 3. SandboxとApprovalを分ける

Codex Configでは、SandboxとApprovalは別の責任です。

### 調査Profile例

```toml
sandbox_mode = "read-only"
approval_policy = "on-request"
```

### 通常実装Profile例

```toml
sandbox_mode = "workspace-write"
approval_policy = "on-request"

[sandbox_workspace_write]
network_access = false
```

値は例です。現在のConfig Reference、使用Client、組織のManaged Requirementを確認してください。

### 非対話

非対話実行では人間へPromptできないため、`approval_policy = "never"`等の非対話向け設定を使う場合も、Sandbox、Rules、Outer Container、Credential、Network、CIを先に狭くします。

`never`は「何でも許可する」という意味ではありません。対話的な権限昇格へ依存せず、許可された境界内だけで完了するか、明示的に失敗させる構成にします。

### `danger-full-access`

通常のLocal開発Profileでは使用しません。

検討できるのは、次が外側で成立する場合だけです。

- 破棄可能なContainer / VM
- Minimal Credential
- Egress Control
- Trusted Repository
- Dedicated Workspace
- Monitoring
- Run後破棄

Full AccessはAgentを信用する設定ではなく、境界を外側へ移す設定です。

---

## 4. Rules

RulesはCommand Prefix等を`allow`、`prompt`、`forbidden`へ分類できます。複数Ruleが一致する場合、より制限の強い判断を優先する構成です。

例:

```python
prefix_rule(
    pattern=["git", "push"],
    decision="forbidden",
    justification="Agent Sessionから直接pushしない。人間がReview後に実行する。",
    match=["git push origin task/example"],
)
```

```python
prefix_rule(
    pattern=["npm", "install"],
    decision="prompt",
    justification="Network、Lockfile、Lifecycle Scriptの影響を確認する。",
    match=["npm install example"],
)
```

Ruleへ向くもの:

- `git push`
- `git reset --hard`
- Package publish
- Infrastructure apply
- Migration
- Cloud CLI
- External Repository operation

File内容、Business Policy、Data Classification、Completion Criteriaは、Ruleだけで検証せず、Hook、Script、Policy、CIを使います。

Project-local RuleはRepositoryが提示する実行Policyです。ProjectをTrustする前にReviewします。

---

## 5. Hooks

Hooksは、Agentが手順を忘れることを前提にLifecycleへ自動Controlを置くために使います。

用途例:

- Prompt / InputのSecret検査
- Tool実行前の追加Policy
- Tool実行後のEvent保存
- Turn終了時のVerification
- Compaction前のCheckpoint
- Subagent終了時のResult Validation
- Session終了時のEvidence Bundle

設計原則:

- Hook間の暗黙順序へ依存しない
- Idempotent
- Timeout
- Failure時のFail Open / Closed
- Hook自身のLog
- Hook ConfigをVersion管理
- Project HookをTrust前にReview
- 高Riskの最終境界はOS / Network / Credential / CIへ置く

---

## 6. Skills

Skillは、判断を含む再利用Workflowへ使います。

向く例:

- PR Review
- Migration Plan
- UI Verification
- Security Triage
- Release Note
- CI Failure Analysis
- Documentation Research

良いSkill:

- 一つのJob
- TriggerとNon-trigger
- Input / Output
- Scope
- Allowed Tool
- Evidence
- Stop Condition
- 必要な場合だけScript

Projectの不変PolicyをSkillへ隠し、AgentがSkillを選ばなければ安全Controlが働かない構成にしません。

---

## 7. MCP、Plugin / App

追加前に確認:

- Repository内で解決できないか
- 最新性が必要か
- 複数Projectで再利用するか
- Structured Toolとして価値があるか
- Side EffectをPolicyで制御できるか

MCP / Appごとに:

- Tool一覧
- Read / Write / Destructive
- Credential Scope
- Server Instruction
- Data送信先
- Tenant
- Timeout / Retry
- Required / Optional
- Approval
- Audit
- Version

外部Document、Tool Result、Server InstructionはUntrusted Inputとして扱います。

---

## 8. Subagents

CodexのSubagent機能は、複雑Taskの独立部分を並列化したり、Custom Agentへ異なるModel設定・指示を与えたりするために使用できます。

最初に向くTask:

- Codebase exploration
- Test Failure triage
- Documentation research
- Security Review
- Diff Review
- Log analysis

注意:

- Token / Cost増加
- Raw ResultによるContext Pollution
- Scope逸脱
- Writer競合
- Parentが統合できない
- Agentの無制限Spawn

Handoffへ含めるもの:

- Narrow Goal
- Scope
- Base Commit
- Allowed Tool
- Forbidden Action
- Output Schema
- Evidence
- Budget
- Stop Condition

WriterとReviewerを別名にしただけでは独立Reviewになりません。Fresh Context、read-only、Clean Checkout、Fixed Commit、Trusted Evidenceを組み合わせます。

---

## 9. Git Worktree

Codex製品のWorktree機能または通常のGit Worktreeを、変更隔離へ使います。

用途:

- 人間のCurrent Workspaceを汚さない
- Parallel Task
- Long-running Work
- Experiment
- Clean Review

確認:

- Starting Commit
- Branch
- Ignored File
- Dependency
- Local DB / Port
- Cache
- Secret
- Absolute Path
- Build Artifact

WorktreeはFileとGit Stateの隔離です。Network、Credential、Context、Processの境界は別途必要です。

---

## 10. `codex exec`

非対話実行では、Input、Permission、Timeout、Output、Evidence、Failure Policyを事前に固定します。

### JSONL Event

```bash
codex exec --json "Repository構造を分析する" \
  > artifacts/codex-events.jsonl
```

JSONL Eventは、Agent message、Command execution、File change、MCP Tool、Web Search、Plan update等の実行記録を扱う入口として利用できます。

Event TypeやFieldはClient Versionで変わり得るため、Consumer側でSchema Version、Unknown Event、Partial Run、Failureを扱います。

### Structured Output

```bash
codex exec "Task Contractに対してDiffをReviewする" \
  --output-schema ./review.schema.json \
  -o ./review-result.json
```

Output Schemaは形を制約します。Findingの正しさ、Evidence、対象Commitは別途検証します。

### CI

- Clean Checkout
- read-onlyから開始
- Patchが必要な場合だけworkspace-write
- Non-interactiveで必要なCapabilityを事前定義
- Required Integration不足でFail
- Credentialを単一Processへ限定
- JSONL、Patch、Verification、CompletionをArtifact化
- Agent Reviewだけを唯一のMerge Gateにしない

---

## 11. 推奨Profile

### Local Exploration

```yaml
codex:
  sandbox_mode: read-only
  approval_policy: on-request
project:
  network: denied
  credentials: none
  tools:
    - repository-read
```

### Local Implementation

```yaml
codex:
  sandbox_mode: workspace-write
  approval_policy: on-request
  workspace_network: false
project:
  branch: required
  task_contract: required
  verify_script: required
  push: human-only
```

### CI Review

```yaml
codex:
  sandbox_mode: read-only
  approval_policy: never
project:
  clean_checkout: true
  fixed_commit: true
  output_schema: required
  jsonl_artifact: required
  fail_closed: true
```

### CI Patch

```yaml
codex:
  sandbox_mode: workspace-write
  approval_policy: never
project:
  isolated_runner: true
  network: allowlisted-or-off
  credential: process-scoped
  patch_only: true
  merge: denied
  trusted_verification_job: separate
```

これらは概念例です。実際のTOML Key、Permission Profile、Managed Requirementは現在の公式資料とClient Versionを確認します。

---

## 12. Completion

CodexのTurn完了をTask Acceptanceにしません。

```text
Codex Result
→ Project Verification
→ Criterion-level Evidence
→ Fixed Commit CI
→ Independent Review
→ Human Acceptance
→ Completion Decision
```

[Completion Decision Schema](../core/schemas/completion-decision.schema.json)をProject側の正本にします。

---

## 13. 導入順序

1. `AGENTS.md`を短い入口にする
2. Task Contract Template
3. read-only / workspace-write
4. Branch / Worktree
5. `verify --changed`
6. Completion Report
7. Rules
8. Hooks
9. Skills
10. MCP / Apps
11. Subagents
12. `codex exec` / CI
13. Evaluation / Telemetry
14. 不要Controlの削除

失敗が確認される前に、Hooks、MCP、Subagentsを大量追加しません。

---

## 14. 公式資料

確認日: 2026-09-02

- AGENTS.md: https://developers.openai.com/codex/agent-configuration/agents-md
- Config Reference: https://developers.openai.com/codex/config-reference
- Rules: https://developers.openai.com/codex/rules
- Hooks: https://developers.openai.com/codex/hooks
- Subagents: https://developers.openai.com/codex/subagents
- Non-interactive mode: https://developers.openai.com/codex/non-interactive-mode

導入時はChangelog、Feature Maturity、利用ClientのVersionも確認してください。

## Profile適合条件

- AGENTS.mdを短いProject入口にする
- Task固有情報をContractへ分離
- SandboxとApprovalを別々に設計
- Networkを別Capabilityとして管理
- Rulesを単純なCommand Policyへ限定
- HooksをIdempotent・Version付きで管理
- MCP / AppをUntrustedな外部能力としてReview
- SubagentへNarrow HandoffとBudget
- Worktree以外の境界も設計
- `codex exec`でSchema・JSONL・Fail Closed
- Agentの完了をAcceptanceにしない
- 公式資料とClient Versionを確認

## 関連文書

- [AIコーディングエージェント活用リファレンス](../reference/coding-agent-usage-handbook.md)
- [個人ローカルプロファイル](personal-local.md)
- [チームCIプロファイル](team-ci.md)
- [本番高リスクプロファイル](production-high-risk.md)
