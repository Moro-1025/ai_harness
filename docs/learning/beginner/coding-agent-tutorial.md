---
title: "初級: AIコーディングエージェントで小さなBugを安全に修正する"
version: "1.2"
core_version: "1.0"
audience: "beginner"
risk_scope: "low-to-medium"
updated: "2026-09-02"
---

# AIコーディングエージェントで小さなBugを安全に修正する

## このチュートリアルで行うこと

小さなBug修正を、次の流れで実施します。

```text
Task Contract
→ Baseline
→ read-only調査
→ Plan確認
→ Branchで実装
→ Targeted Verification
→ Diff Review
→ CriterionごとのEvidence
→ Completion Report
```

製品固有の操作ではなく、Codex、Claude Code、Cursor系、その他のコーディングエージェントへ共通するProject Harnessを扱います。Codex固有設定は[Codexプロファイル](../../profiles/codex.md)を参照してください。

### 想定Task

> 期限切れRefresh Tokenを受け取ったAPIが、本来401を返すべきところ500を返す不具合を修正する。

### 対象外

- 本番反映
- DB Migration
- Public API形式変更
- 新しい外部依存
- Credential利用
- Release、Push、Mergeの自動化

これらが必要だと分かった時点で停止します。

---

## 1. 開始状態を確認する

Agentへ依頼する前に、人間またはBootstrap Scriptで確認します。

```bash
git status --short
git branch --show-current
git rev-parse HEAD
```

Project固有のCommandでBaseline Testも実行します。

```bash
scripts/agent/verify --scope auth
```

まだ`verify` Scriptがない場合は、既存のTest Commandを一つの入口へまとめることを最初の改善Taskにします。

Baselineが既に壊れている場合は、新しいBugと区別できるよう結果を保存し、修正Taskを続けるか人間が判断します。

---

## 2. Task Contractを書く

`task.md`:

```markdown
# Task Contract

## 目的

期限切れRefresh Tokenで500になる問題を修正する。

## 期待する状態

期限切れTokenは401を返し、有効Tokenの既存動作は変えない。

## 変更可能範囲

- `src/auth/**`
- `tests/auth/**`

## 変更禁止範囲

- `migrations/**`
- Public API Schema
- Dependency定義
- CI設定

## Non-goal

- 認証基盤全体のRefactor
- Error形式の統一
- Performance改善

## 完了条件

- AUTH-001: 期限切れTokenで401を返す
- AUTH-002: 有効Tokenの既存Testが成功する
- AUTH-003: DB書込みが発生しない
- SCOPE-001: 許可範囲外の変更がない

## 必要な証拠

- 変更前の再現Test
- 変更後のTargeted Test
- 関係する既存Test
- `git diff --check`
- 変更File一覧とDiff
- 未実行Testと残存Risk

## Human Gate

次が必要なら実装を止める。

- Public API変更
- DB Migration
- 新しいDependency
- Scope外の設計変更
- Credentialまたは本番接続

## Stop Condition

- Baselineの原因不明な失敗
- 同一Errorを2回繰り返す
- 指定範囲外の変更が必要
- 検証Environmentが不足
```

ここで数値は説明用です。実際のRepositoryに合わせて調整します。

---

## 3. 調査だけを依頼する

最初は書込みを許可しません。

依頼例:

```text
task.mdを契約として扱ってください。

まずread-onlyでRepositoryを調査し、次だけを返してください。

1. 500が発生する実行経路
2. 関係するFileとSymbol
3. 最小変更案
4. 追加・更新するTest
5. Task Contractを変更する必要がある点
6. 停止条件へ該当する事項

まだFileを変更しないでください。
```

調査結果を確認します。

- 実際のCodeを根拠にしているか
- Writerの推測だけになっていないか
- Scope内で修正できるか
- Testが現象を直接再現するか
- 大きなRefactorへ広がっていないか

ScopeやCompletion Criteriaが変わる場合、暗黙に続けずTask Contractを改訂します。

---

## 4. Branchを作る

```bash
git switch -c fix/expired-refresh-token
```

人間の未Commit変更がある、別Taskを並列に進める、CleanなReview環境が必要な場合はWorktreeを使います。

```bash
git worktree add ../worktrees/expired-token -b fix/expired-refresh-token
```

原則は`1 Task = 1 Branch = 1 Completion Report`です。

---

## 5. 実装を依頼する

実装時だけWorkspace内の書込みを許可します。Networkは原則無効です。

依頼例:

```text
承認したPlanに従い、task.mdの範囲だけを変更してください。

実装規則:
- まず失敗を再現するTestを追加または確認する
- 修正は必要最小限にする
- Public API、DB Schema、Dependencyは変更しない
- 各変更後に対象Testを実行する
- Scope外が必要なら停止する
- 既存の未関連差分をRevertしない

最後にCompletion Reportを作成してください。
```

「きれいにして」「関連箇所も改善して」のような無制限の追加改善を許可しません。見つけた改善候補は、現在Taskを完了した後の次Task候補として報告させます。

---

## 6. 検証する

推奨順序:

```text
再現Test
→ 変更に近いUnit Test
→ Auth Module Test
→ 関係するIntegration Test
→ 静的検査
→ Diff Review
```

Command例:

```bash
scripts/agent/verify --scope auth
git diff --check
git status --short
git diff -- src/auth tests/auth
```

Full Suiteが必要かはTaskのRiskとRepositoryの規約で決めます。実行できないTestを成功扱いしてはいけません。

---

## 7. CriterionとEvidenceを対応付ける

Completion Reportの前に、次のMatrixを作ります。

```markdown
| Criterion | Status | Verifier | Subject | Evidence |
|---|---|---|---|---|
| AUTH-001 | pass | integration-test | commit候補の現在Diff | tests/auth/expired-token.log |
| AUTH-002 | pass | existing-test-suite | 現在Diff | tests/auth/valid-token.log |
| AUTH-003 | pass | DB spy assertion | 現在Diff | tests/auth/no-write.log |
| SCOPE-001 | pass | git diff | starting commit..working tree | diff.patch |
```

Agentが「全部通りました」と書いただけでは不十分です。Command結果とDiffを実際に確認します。

---

## 8. DiffをReviewする

人間またはFresh ContextのReviewerへ、次だけを渡します。

- Task Contract
- Starting Commit
- 実際のDiff
- Test結果
- Architecture上の短い規則
- 既知のRisk

Writerの長い説明を先に読ませると、Reviewerが同じ前提へ引っ張られることがあります。

確認観点:

- 500の根本原因を直しているか
- Errorを握りつぶして401に見せていないか
- 有効Token経路を壊していないか
- Auth以外を変更していないか
- Testが実装詳細ではなく要求を検証しているか
- 過剰な抽象化や不要なDependencyがないか
- 未検証経路が明示されているか

---

## 9. Completion Reportを作る

```markdown
# Completion Report

## Summary

期限切れRefresh TokenのError mappingを修正し、401を返すようにした。

## Completion Criteria

| Criterion | Result | Evidence |
|---|---|---|
| AUTH-001 | pass | `artifacts/auth-expired.log` |
| AUTH-002 | pass | `artifacts/auth-valid.log` |
| AUTH-003 | pass | `artifacts/auth-no-write.log` |
| SCOPE-001 | pass | `artifacts/diff.patch` |

## Files Changed

- `src/auth/refresh.py`
- `tests/auth/test_refresh.py`

## Commands

```text
scripts/agent/verify --scope auth
git diff --check
```

## Not Run

- Full E2E: Local環境に外部Identity Providerがないため未実行

## Residual Risk

- 実ProviderとのE2EはCIで確認が必要

## Rollback

Branchを破棄するか、採用後は対象CommitをRevertする。
```

Completion ReportはEvidenceへの索引です。Report内の自己説明だけで採用しません。

---

## 10. 採用判断

最低限、次を確認します。

```text
すべての必須Criterionがpass
AND
Evidenceが現在のDiffまたはCommitへ対応
AND
Scope外変更なし
AND
重大Errorなし
AND
未実行項目と残存Riskを報告
```

一つでも成立しない場合は、`accepted`ではなく`rejected`、`blocked`、`needs_human`のいずれかです。

---

## 11. 失敗時の扱い

### Scope外変更が必要

実装を止め、Task Contract改訂案を返します。勝手にDependencyやMigrationを追加しません。

### Testを実行できない

理由、必要Environment、代替Evidence、残存Risk、人間が実行する手順を報告します。

### 同じ修正と失敗を繰り返す

追加Promptを重ねる前に停止し、仮説、試行、Evidence、未解決点をまとめます。

### Agentが余計な改善を行った

採用せず、Task変更と追加改善を分離します。Scope内Patchだけを取り出せない場合はBranchを破棄して再実行します。

---

## 12. 完了条件

- Baselineを記録した
- Task Contractに成果、Scope、Criterion、Evidence、停止条件がある
- read-only調査を実装前に行った
- BranchまたはWorktreeで変更を隔離した
- NetworkとCredentialを不要のまま維持した
- CriterionごとにEvidenceを対応付けた
- `git diff`をReviewした
- 未実行項目と残存Riskを報告した
- Agentの自己申告だけで採用していない

次は[チームハーネス](../intermediate/team-harness.md)へ進んでください。

## 参照

- [中核設計原則](../../core/principles.md)
- [統合判定モデル](../../core/decision-model.md)
- [個人ローカルプロファイル](../../profiles/personal-local.md)
- [Codexプロファイル](../../profiles/codex.md)
