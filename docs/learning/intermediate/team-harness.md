---
title: "中級: 個人運用をチームハーネスへ拡張する"
version: "1.2"
core_version: "1.0"
audience: "intermediate"
risk_scope: "low-to-high"
updated: "2026-09-02"
---

# 個人運用をチームハーネスへ拡張する

## ゴール

個人がAIエージェントをうまく使えた状態と、チームが安全・再現可能に運用できる状態は異なります。

この文書では、個人のPromptや暗黙知を、Repository、Task Contract、権限Profile、Verification、Trusted CI、Review、Incident運用へ移します。

```text
個人の成功例
→ 共有可能なTask Contract
→ Version管理されたProject Harness
→ 共通Verification
→ 独立したAcceptance
→ IncidentとEvalによる改善
```

---

## 1. チーム化で増える失敗

個人では気付きにくい失敗が増えます。

- 人によってTaskの書き方が異なる
- Agent製品・Version・設定が異なる
- Testの選び方が毎回変わる
- Scope外変更が混ざる
- Evidenceの保存場所が不明
- WriterとReviewerが同じ前提を共有する
- CIとLocalのEnvironment差で結果が変わる
- Prompt、Skill、Hook、Ruleが重複する
- Policy変更がReviewされない
- Agentの失敗が学習資産にならない
- Human Gateの責任者が不明
- 高リスクTaskが個人のFull Accessで実行される

チーム化の目的はAgentを自由に増やすことではなく、**誰が実行しても同じ境界と完了判定を使うこと**です。

---

## 2. 所有するものを決める

| 対象 | 推奨Owner |
|---|---|
| 中核設計原則・用語 | Architecture / Platform |
| Task Contract Template | 各Domain Owner + Platform |
| Capability Profile | Security / Platform |
| Project指示 | Repository Maintainer |
| Verification Script | Repository Maintainer |
| Trusted CI | Platform / CI Owner |
| Policy | Security / Domain Owner |
| Acceptance | Product / Code Owner |
| Incident対応 | On-call / Security |
| Eval Dataset | Platform + Domain Experts |

AgentにOwnerを置きません。最終責任は人間または組織のRoleへ割り当てます。

---

## 3. Repositoryへ置く最小構成

例:

```text
repo/
├─ AGENTS.md
├─ docs/
│  └─ agent/
│     ├─ TASKS.md
│     ├─ PLANS.md
│     ├─ TESTING.md
│     ├─ CODE_REVIEW.md
│     ├─ SECURITY.md
│     └─ INCIDENTS.md
├─ scripts/
│  └─ agent/
│     ├─ bootstrap
│     ├─ doctor
│     ├─ verify
│     ├─ smoke
│     └─ collect-evidence
├─ .agent/
│  ├─ task-template.yaml
│  ├─ schemas/
│  ├─ policies/
│  └─ evals/
└─ .agent-runs/       # 原則gitignore
```

すべてを最初から追加しません。

### 段階1

```text
AGENTS.md
+ Task Template
+ Branch
+ verify Script
+ Completion Report
```

### 段階2

```text
権限Profile
+ Trusted CI
+ Evidence Bundle
+ Fresh Reviewer
```

### 段階3

```text
Policy Decision
+ Approval Record
+ Eval Dataset
+ Incident Runbook
```

---

## 4. AGENTS.mdを短い入口にする

`AGENTS.md`へすべてを書きません。

含めるもの:

- Repositoryの入口
- Build、Test、Lint、Formatの正しいCommand
- 変更禁止範囲
- Architecture上の重要境界
- 完了前に必ず行うVerification
- 詳細文書を読むTrigger

含めないもの:

- 一度だけのTask情報
- 長い設計説明
- API仕様全文
- Secret
- 機械的に強制できる禁止事項の長文
- 特定Model向けの一時的回避策

例:

```markdown
# Repository guidance

- Source: `src/`
- Tests: `tests/`
- Generated files: `generated/`。直接編集しない。
- Setup: `scripts/agent/bootstrap`
- Verify changed scope: `scripts/agent/verify --changed`
- Public API、DB Schema、Dependencyを変更する前に停止する。
- 複数Subsystem変更では`docs/agent/PLANS.md`を読む。
- Security境界変更では`docs/agent/SECURITY.md`を読む。
- 完了前にDiff、Test、未実行、残存Riskを報告する。
```

同じ失敗を防ぐ場所は原因で選びます。

| 原因 | 主な改善先 |
|---|---|
| Task固有の制約不足 | Task Contract |
| 常に必要な短い規則 | AGENTS.md |
| 繰り返す判断手順 | Skill |
| 決定論的な処理 | Script |
| Lifecycleでの自動確認 | Hook |
| Commandの許可・禁止 | Rule |
| 外部Tool・Data | MCP |
| 越えてはならない境界 | OS、Network、Credential、CI、Git保護 |

---

## 5. Task Contractをチーム契約にする

最低限の項目:

```text
目的
期待状態
Scope / Forbidden Scope
Non-goal
Constraints
Completion Criteria
CriterionごとのEvidence
Risk Assessment
Capability Profile
Human Gate
Execution Budget
Stop Conditions
Contract Version / Hash
```

Task ContractはIssue本文だけに閉じず、Run ArtifactまたはVersion管理されたFileとして保持します。

### 変更手続き

調査中に契約を変える場合:

1. 現在の実行を安全な地点で停止する
2. 変更理由とField差分を書く
3. Risk Assessmentを更新する
4. CapabilityとHuman Gateを再計算する
5. 必要な人へ再承認を求める
6. 新しいContract Hashを発行する
7. 旧Evidenceを再利用できるかCriterion単位で判断する
8. 旧契約を`superseded`として保持する

Scopeだけ変えてContract Versionを据え置く運用をしません。

---

## 6. Taskの大きさを制御する

Task分割の基準:

```text
独立して検証できるか
AND
独立してReviewできるか
AND
独立してRollbackできるか
```

大きすぎる兆候:

- 完了条件を一文で説明できない
- 独立した変更理由が複数ある
- 多数のSubsystemへ同時に触れる
- Human Gateが複数Domainへまたがる
- Rollback単位が不明
- Reviewerの専門性が一人では足りない
- Evidenceの対象Versionを固定できない

複雑なTaskでは、Planをレビュー可能な成果物にします。

---

## 7. 権限ProfileをTask種別ごとに定義する

例:

| Profile | File | Network | Credential | Side Effect | Human Gate |
|---|---|---|---|---|---|
| exploration | read-only | off | none | none | 原則不要 |
| code-review | read-only |限定検索 | none | none | 重要Finding採用時 |
| implementation | workspace-write | off | none | local only | Scope変更時 |
| dependency-update | workspace-write | registry allowlist | ephemeral | lockfile | Install前 |
| ci-review | read-only |最小 | single-process | none | 非対話Fail Closed |
| release | dedicated | allowlist | short-lived | production | 必須 |

Profile名だけで安全を保証しません。実際のSandbox、Network、Credential、Tool Allowlistを機械的に確認します。

### RiskとProfileを分ける

同じ`read-only`でも、Public Sourceだけを読むTaskと顧客Dataを読むTaskではRiskが異なります。

```text
Risk Assessment
→ 必要なControlを決める
→ Capability Profileを選ぶ
→ Policyが今回のActionを判定する
```

---

## 8. Gitを隔離・採用の境界にする

基本単位:

```text
1 Task
1 BranchまたはWorktree
1 Contract Hash
1 Evidence Bundle
1 Completion Decision
```

開始時に保存:

- Starting Commit
- Branch / Worktree
- `git status`
- 既存差分
- Baseline Test
- Agent Product / Version
- Capability Profile

終了時に保存:

- Ending CommitまたはPatch Hash
- 変更File
- Diff
- Verification Results
- Completion Report
- Reviewer Findings
- Trusted CI Run

### 並列化

Read-heavyな調査は同じSnapshotで並列化できます。Write-heavyなTaskは、Worktree、Branch、File Ownershipを分離します。

同じWorking Treeへの複数Writerを標準構成にしません。

---

## 9. Verificationの入口をProject側で持つ

「必要なTestを実行してください」だけでは実行内容が変わります。

```text
scripts/agent/verify --scope auth
scripts/agent/verify --changed
scripts/agent/verify --full
```

良いScript:

- 非対話
- 正しいExit Code
- 失敗理由が短い
- Log出力先が一定
- 変更対象から検証Scopeを導出できる
- Version情報を記録する
- SecretをMaskする
- Machine-readableなSummaryを返す

例:

```json
{
  "status": "pass",
  "subject_commit": "abc123",
  "checks": [
    {
      "criterion_id": "AUTH-001",
      "status": "pass",
      "evidence_ref": "artifacts/auth-expired.xml"
    }
  ],
  "warnings": [],
  "not_run": []
}
```

---

## 10. Evidenceを二層以上に分ける

### Agent実行Evidence

Writerが作業中に行う自己検証です。速い一方、Test選択、Workspace、保存先をWriterが支配できます。

### Trusted Evidence

固定Commitを、Writerが変更できないCI定義・権限・Artifact Storeで検証します。

中〜高Riskでは、次を分けます。

```text
Writerの自己検証
+
Project verify Script
+
Trusted CI
+
独立Reviewer
+
Human Acceptance
```

すべてのTaskへ全層を要求するのではなく、RiskとCriterionに応じて必要な独立性を決めます。

---

## 11. Reviewerを独立させる

最低限:

- Fresh Context
- read-only
- Starting Commitと対象Commitを固定
- Clean Checkoutまたは独立Worktree
- Task Contractと実際のDiffを正本にする
- Writerの説明だけを根拠にしない
- FindingへFile、Line / Symbol、Reason、Evidenceを要求する
- Reviewと修正を同時に行わせない

高Riskでは、異なるModel、Trusted CI、Security専門家、人間Reviewを組み合わせます。

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

---

## 12. AcceptanceのOwnerをVendor外へ置く

Agent製品の`completed`や終了Codeを、そのまま採用判定にしません。

```text
必須CriterionのVerification
+ Evidence Binding
+ Scope Check
+ Policy Check
+ Approval Check
+ Uncertain Actionなし
+ Residual Risk
→ Completion Decision
```

機械的な判定はTrusted CIまたはProject側のAcceptance Serviceが行い、人間はRequirement、Architecture、UX、Risk Acceptanceへ集中します。

---

## 13. Session Artifactの方針

例:

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

管理項目:

- Git管理するか
- Local / CI / Audit Storeの保存期間
- 容量上限
- Secret、PII、Source CodeのMasking
- TenantごとのAccess
- Writerが改ざんできる範囲
- 正本は何か
- 削除要求・監査保全との両立

`.agent-runs/`だけを高Risk Taskの唯一の監査証拠にしません。

---

## 14. IncidentをHarness改善へ戻す

初動:

```text
停止
→ 外部副作用確認
→ Branch / Worktree隔離
→ 必要ならCredential失効
→ Evidence保全
→ RollbackまたはCompensation
→ 原因分類
→ Control改善
→ Eval Case追加
```

失敗分類:

- Requirement
- Context
- Environment
- Permission
- Tool
- Change
- Verification
- Review
- Coordination
- Security
- Automation
- Policy
- Approval

改善後は、同じ代表Taskで変更前後を比較します。新しいPrompt規則を増やすだけでなく、Script、Policy、Permission、CIなど、原因に最も近い境界を修正します。

---

## 15. 導入ロードマップ

### Sprint 1: 共通入口

- Task Template
- 短い`AGENTS.md`
- Branch運用
- `verify --changed`
- Completion Report

### Sprint 2: 証拠と隔離

- Evidence Bundle
- Worktree運用
- read-only / workspace-write Profile
- Fresh Reviewer
- Fixed Commit CI

### Sprint 3: Policy

- Risk Assessment
- Capability Profile
- Policy Decision記録
- Approval Record
- Contract Amendment

### Sprint 4: Evaluation

- Eval Dataset Version
- Incident Case
- Task種別ごとのMetrics
- Harness変更Release Gate
- Control削除Review

各Sprintで実際の失敗を確認し、不要な機構を先回りして追加しません。

---

## 16. チーム導入の完了条件

- Core VersionとProfileを明示している
- Task Contractの正本と改訂手続きがある
- Task種別ごとのCapability Profileがある
- `verify`の共通入口がある
- Branch / WorktreeでWriterを隔離する
- CriterionとEvidenceを対応付ける
- Trusted CIが固定Commitを検証する
- Reviewerの独立性を定義している
- Agentの自己申告以外でAcceptanceを判断する
- Session ArtifactのRetentionとSecret方針がある
- IncidentをEvalへ戻す
- 古いControlを削除するReviewがある

## 参照

- [中核設計原則](../../core/principles.md)
- [統合判定モデル](../../core/decision-model.md)
- [証拠とポリシー](evidence-and-policy.md)
- [CI・レビュー・評価](ci-review-and-evaluation.md)
- [チームCIプロファイル](../../profiles/team-ci.md)
