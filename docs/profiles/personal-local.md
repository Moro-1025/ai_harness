---
title: "Profile: 個人ローカル開発"
version: "1.0"
core_version: "1.0"
profile_id: "personal-local"
risk_scope: "low-to-medium"
updated: "2026-09-02"
---

# Profile: 個人ローカル開発

## 目的

個人がローカル環境でAIエージェントまたはAIコーディングエージェントを使うときの、最小で安全な既定値です。

このProfileは、[中核設計原則](../core/principles.md)を弱めません。高リスク操作を簡単に実行するためのProfileではありません。

---

## 適用範囲

適するTask:

- Public / InternalなRepositoryの調査
- Local Fileの要約
- 小さなBug修正
- Test追加
- Documentation
- Local-only Prototype
- Branch内の可逆な変更
- 人間が全Diffを確認できる規模

対象外:

- Production
- 顧客Data、医療・金融等のRestricted Data
- 長期Credential
- 外部送信
- Public公開の自動実行
- 金銭
- 権限変更
- 大量削除
- 法的確定
- 複数Worker
- Human ReviewなしのMerge / Release

対象外が必要になった場合、[本番高リスクプロファイル](production-high-risk.md)または[分散実行プロファイル](distributed-execution.md)へ移行します。

---

## 既定Capability

```yaml
profile_id: personal-local
filesystem:
  exploration: read-only
  implementation: workspace-write
network:
  default: denied
  exception: task-specific-allowlist
credential_access: none
external_side_effects: denied
allowed_data_classifications:
  - public
  - internal
human_gate:
  before_scope_expansion: required
  before_dependency_install: required
  before_external_action: required
```

### 原則

- 調査はread-onlyから開始する
- 実装時だけworkspaceへ書く
- Home DirectoryやRepository外を許可しない
- Networkは必要なTaskだけ有効化する
- CredentialをPrompt、Context、Repositoryへ置かない
- `push`、`merge`、`release`はAgent実装Taskから分離する
- Full Accessを通常運用にしない

---

## 最小Repository構成

```text
repo/
├─ AGENTS.md
├─ scripts/
│  └─ agent/
│     └─ verify
├─ task.md
└─ .agent-runs/     # gitignore
```

### AGENTS.md

短く保ちます。

```markdown
- 変更前に`task.md`を読む。
- 調査中はFileを変更しない。
- 変更は現在BranchとTask Scope内だけ。
- Dependency、Public API、DB Schemaの変更前に停止する。
- 完了前に`scripts/agent/verify --changed`を実行する。
- Test未実行、残存Risk、変更Fileを報告する。
```

### Task Contract

最低限:

- 目的
- 期待状態
- Scope / Forbidden
- Completion Criteria
- Evidence
- Stop Condition

---

## 実行フロー

```text
1. `git status`とStarting Commitを記録
2. Task Contractを書く
3. read-onlyで調査
4. Planを人間が確認
5. Branch / Worktreeを作る
6. workspace-writeで最小変更
7. Targeted Verification
8. `git diff --check`
9. 人間が全Diffを確認
10. Completion Report
```

### Branch

```bash
git switch -c task/example-change
```

人間の未Commit作業がある、複数Taskを並列に進める場合はWorktreeを使います。

---

## Evidence

最低限:

- Starting Commit
- File List
- Diff
- 実行Command
- Test Result
- CriterionごとのPass / Fail / Not Run
- 未実行項目
- 残存Risk
- Rollback方法

Local ArtifactはWriterが変更できるため、高Risk Evidenceにはなりません。このProfileでは人間が全DiffとTestを確認することを前提とします。

---

## Stop Conditions

- Scope外が必要
- Public API、DB Schema、Dependency変更が必要
- Credentialが必要
- Network送信が必要
- Baselineが原因不明で失敗
- 同じErrorを繰り返す
- Test Environment不足
- Humanが全Diffを確認できない規模
- Restricted Dataを扱う必要
- Productionへの接続が必要

---

## Profile適合条件

- read-onlyとworkspace-writeを分けた
- NetworkはDefault Deny
- CredentialをAgentへ渡していない
- BranchまたはWorktreeを使う
- Task Contractがある
- `verify`の入口がある
- 人間が全Diffを確認する
- Agentの自己申告だけで完了にしない
- Scope変更で停止する
- 外部送信・Productionを含まない

## 関連文書

- [初級: ハーネスの基礎](../learning/beginner/harness-basics.md)
- [コーディングエージェントのチュートリアル](../learning/beginner/coding-agent-tutorial.md)
- [Codexプロファイル](codex.md)
