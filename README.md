# AI Harness Handbooks

AIエージェントとAIコーディングエージェントを、モデル単体ではなく、**権限・状態・検証・証拠・停止条件を含むシステム**として設計・運用するための日本語ドキュメントです。

このリポジトリでは、設計原則を一つの正本へ固定し、学習用ガイド、実装リファレンス、運用プロファイルを分離しています。

## 最初に読む

| 読者・目的 | 入口 |
|---|---|
| 初めてハーネスを学ぶ | [ハーネスの基礎](docs/learning/beginner/harness-basics.md) |
| 小さなAIエージェントを作る | [単一エージェントのチュートリアル](docs/learning/beginner/simple-agent-tutorial.md) |
| Codex等で安全にコード変更する | [コーディングエージェントのチュートリアル](docs/learning/beginner/coding-agent-tutorial.md) |
| チーム・CIへ導入する | [チームハーネス](docs/learning/intermediate/team-harness.md) |
| 設計標準を確認する | [中核設計原則](docs/core/principles.md) |
| 高度な設計を参照する | [AIエージェント開発リファレンス](docs/reference/agent-development-handbook.md) |
| Codex固有の対応を確認する | [Codexプロファイル](docs/profiles/codex.md) |

全体の案内は[ドキュメントマップ](docs/README.md)を参照してください。

## 文書の4層

```text
Normative Core
  └─ 変えてはいけない設計原則、用語、判定モデル、JSON Schema

Learning Guides
  └─ 初級・中級者が順番に学び、手を動かすための説明

Reference
  └─ 高度な設計・実装・運用を調べるための体系的な参照文書

Profiles
  └─ 個人、チームCI、高リスク本番、分散実行、Codexへの適用
```

## 重要な区別

**読者の習熟度**と**システムのリスク**は別の軸です。

```text
習熟度: 初級 / 中級 / 上級
リスク: 低 / 中 / 高 / 重大
```

初級者向けガイドは、高リスク処理を簡単な統制で実行するためのものではありません。初級者向けの対象は、外部副作用、個人情報、認証情報、金銭、本番操作を含まない範囲へ限定します。高リスクな処理は、[本番高リスクプロファイル](docs/profiles/production-high-risk.md)へ移行してください。

## 設計原則の扱い

[中核設計原則](docs/core/principles.md)を本シリーズの規範的な正本とします。Learning、Reference、Profileは原則を説明・適用できますが、弱めたり上書きしたりできません。

特に次を固定します。

- モデルとエージェントを同一視しない
- LLMは操作を提案し、決定論的な境界が許可する
- 作業ではなく成果を定義する
- エージェントの自己申告より証拠を優先する
- 最小権限、実行予算、停止条件を持つ
- 不可逆な操作の直前に人間を置く
- 実際の失敗が確認されてから複雑性を追加する
- 不要になった制御を削除できるようにする

## 旧版について

次のv1.1文書は、履歴と互換性のためルートに残します。

- [AIエージェント開発ハンドブック v1.1](AI_Agent_Development_Handbook_JA.md)
- [AIコーディングエージェント活用ハーネス・ハンドブック v1.1](AI_Coding_Agent_Usage_Harness_Handbook_JA.md)

新しい導線と共通仕様は`docs/`以下を正本とします。旧版を参照する場合も、中核用語と判定モデルは`docs/core/`を優先してください。

## バージョン

- ドキュメントセット: 1.2
- Normative Core: 1.0
- 更新日: 2026-09-02
