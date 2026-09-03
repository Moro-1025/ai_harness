---
title: "AI Harness ドキュメントマップ"
version: "1.2"
core_version: "1.0"
updated: "2026-09-02"
---

# ドキュメントマップ

このディレクトリは、共通原則を一度だけ定義し、読者の習熟度とシステムのリスクに応じて適用方法を分ける構成です。

## 1. Normative Core

実装・製品・読者レベルに依存しない正本です。

| 文書 | 役割 |
|---|---|
| [principles.md](core/principles.md) | 変更してはならない14の中核設計原則 |
| [glossary.md](core/glossary.md) | シリーズ全体の用語定義 |
| [decision-model.md](core/decision-model.md) | 契約から完了・監査までの統合判定モデル |
| [schemas/](core/schemas/) | Task Contract、検証、Policy、承認、完了判定の機械可読仕様 |

## 2. Learning Guides

### 初級

順番に読むことを推奨します。

1. [ハーネスの基礎](learning/beginner/harness-basics.md)
2. [単一エージェントのチュートリアル](learning/beginner/simple-agent-tutorial.md)
3. [コーディングエージェントのチュートリアル](learning/beginner/coding-agent-tutorial.md)

初級ガイドでは、外部送信、本番変更、認証情報、個人情報、金銭、不可逆操作を扱いません。

### 中級

1. [チームハーネス](learning/intermediate/team-harness.md)
2. [エージェントランタイム](learning/intermediate/agent-runtime.md)
3. [証拠とポリシー](learning/intermediate/evidence-and-policy.md)
4. [CI・レビュー・評価](learning/intermediate/ci-review-and-evaluation.md)

個人の成功例を、チームとCIで再現可能にするための設計を扱います。

## 3. Reference

| 文書 | 対象 |
|---|---|
| [AIエージェント開発リファレンス](reference/agent-development-handbook.md) | 自作エージェントの設計、実装、評価、運用 |
| [AIコーディングエージェント活用リファレンス](reference/coding-agent-usage-handbook.md) | Repository、Git、CI、権限、Sessionを含む利用ハーネス |

Referenceは高度な論点を調べるための文書です。最初から通読する必要はありません。

## 4. Profiles

ProfilesはCoreを具体的な運用条件へ適用します。Coreの必須原則を弱めることはできません。

| Profile | 使用場面 |
|---|---|
| [personal-local.md](profiles/personal-local.md) | 個人のローカル開発、低〜中リスク |
| [team-ci.md](profiles/team-ci.md) | チーム、Pull Request、Trusted CI |
| [production-high-risk.md](profiles/production-high-risk.md) | 本番、個人情報、外部副作用、金銭、権限 |
| [distributed-execution.md](profiles/distributed-execution.md) | Queue、複数Worker、長時間処理、不確定結果 |
| [codex.md](profiles/codex.md) | Codexの機能への対応付け |

## 適用順序

```text
Coreを読む
→ 対象タスクのRiskを評価する
→ Capability Profileを選ぶ
→ LearningまたはReferenceで実装する
→ Product Profileへ読み替える
→ Evidenceで完了を判定する
```

## 準拠の表し方

各実装またはプロファイルは、少なくとも次を記録します。

```yaml
document_set_version: "1.2"
core_version: "1.0"
profile: "personal-local"
product_profile: "codex"
```

Profile間で競合した場合は、より制限の強い判定を採用します。明示的な例外が必要な場合は、Task Contractの改訂、Risk再評価、期限付き承認、監査記録を要求します。
