---
title: "初級: AIエージェント・ハーネスの基礎"
version: "1.2"
core_version: "1.0"
audience: "beginner"
risk_scope: "low"
updated: "2026-09-02"
---

# AIエージェント・ハーネスの基礎

## この文書のゴール

この文書を読み終えた時点で、次を説明できることを目標にします。

- モデル、エージェント、ハーネスの違い
- なぜ長いPromptだけでは安全にならないか
- Task Contract、権限、検証、証拠、停止条件の役割
- 初めて作る最小ハーネスに何が必要か
- 初級者が扱ってよい範囲と、上位Profileへ移る境界

このガイドは、外部送信、本番変更、個人情報、Credential、金銭、不可逆操作を扱いません。

> **初級者向けと低リスクは同じ意味ではありません。**
> 高リスクなTaskは、説明を簡単にして実行するのではなく、対象外として停止し、[本番高リスクプロファイル](../../profiles/production-high-risk.md)へ移行します。

---

## 1. モデルだけでは仕事は完了しない

LLMは文章やコードを生成できます。しかし、次を自動では保証しません。

- 正しいFileを読んだか
- 許可された範囲だけを変更したか
- Tool引数が安全か
- Testを本当に実行したか
- 結果が現在のVersionに対するものか
- 同じ副作用を二重実行していないか
- どこで停止すべきか
- 完了条件をすべて満たしたか

そこで、モデルの周囲にハーネスを置きます。

```text
利用者
  ↓
Task Contract
  ↓
Agent Loop
  ├─ Model
  ├─ Tool
  ├─ State
  ├─ Permission
  ├─ Verification
  └─ Stop Condition
  ↓
Evidence付きCompletion Report
```

## 2. 3つを分ける

### モデル

入力を読み、次のAction候補や回答を生成します。

### エージェント

モデルの提案を受け、Toolを使い、状態を更新しながら複数StepでTaskを進めます。

### ハーネス

エージェントが何をしてよいか、どこで検証するか、いつ止めるか、何を証拠として残すかを管理します。

モデルが「このFileを削除したい」と提案することと、実際に削除権限があることは別です。

---

## 3. 最小ハーネスの7要素

### 3.1 Task Contract

作業開始前に、何を成立させるかを固定します。

```markdown
# Task Contract

## 目的
READMEからProject名を確認する。

## 期待する状態
Project名を、根拠となるFileと行を示して回答できる。

## 読取り範囲
workspace/

## 書込み範囲
なし

## 完了条件
- PROJECT-001: Project名を回答する
- PROJECT-002: 根拠Fileを示す

## 停止条件
- workspace外の読取りが必要
- 5 Stepを超える
- READMEが存在しない
```

### 3.2 最小権限

調査だけなら書込みを許可しません。

```yaml
filesystem: read
network: denied
external_side_effects: denied
credential_access: none
```

「使わないでください」とPromptへ書くだけでなく、ToolやOS側で能力を与えないことが重要です。

### 3.3 入出力検証

モデルが作ったTool引数を、そのまま実行しません。

確認例:

- Tool名がAllowlistにある
- Pathが`workspace/`内にある
- `..`やSymlinkで外へ出ない
- 必須引数がある
- 返却値が空でない
- Errorを成功として扱っていない

### 3.4 実行予算

無限に試させません。

```yaml
max_steps: 5
max_tool_calls: 5
same_action_limit: 1
same_error_limit: 1
on_exceed: stop
```

数値は説明用です。実際には代表Taskを実行し、必要量を測って決めます。

### 3.5 状態

会話だけに進捗を置きません。

```json
{
  "phase": "verify",
  "completed": ["READMEを読んだ"],
  "pending": ["回答とEvidenceを照合する"],
  "tool_calls": 2,
  "evidence_refs": ["workspace/README.md#L1"]
}
```

### 3.6 検証と証拠

「Project名はsampleです」と答えただけでは完了ではありません。

```text
Criterion PROJECT-001
→ READMEの見出しからProject名を検証
→ Evidence: workspace/README.md#L1

Criterion PROJECT-002
→ 回答にSource参照が含まれることを検証
→ Evidence: completion.json
```

### 3.7 停止条件

止まることは失敗ではなく、安全機能です。

- 権限が足りない
- 対象が見つからない
- 同じErrorを繰り返す
- 予算を超えた
- Scope外が必要
- 結果を検証できない

停止時は、理由、実施済み、未完了、次に必要な判断を報告します。

---

## 4. 最小の実行フロー

```text
1. Task Contractを読む
2. RiskとCapabilityを確認する
3. 次ActionをModelが提案する
4. HarnessがToolと引数を検証する
5. Toolを実行する
6. 結果を検証してStateとEvidenceを更新する
7. Completion Criteriaを照合する
8. 未完了なら次Step、完了ならReport
9. BudgetまたはStop Conditionに達したら停止
```

擬似コード:

```python
def run(task, model):
    state = load_state(task)

    while True:
        enforce_budget(state)

        if all_required_criteria_pass(state):
            return completion_report(state)

        proposal = model.propose(state)
        action = validate_proposal(proposal, task.capabilities)
        result = execute(action)
        verification = verify(action, result, task.criteria)

        record_event(action, result, verification)
        update_state(state, verification)

        if verification.requires_human:
            return stop_and_escalate(state)
```

モデルは`proposal`を作りますが、`validate_proposal`、`execute`、`verify`、`enforce_budget`を自分で無効化できません。

---

## 5. Prompt、Script、Policyの役割

| 目的 | 適した場所 |
|---|---|
| 調査の進め方を伝える | Prompt、Skill |
| JSONの型を確認する | Schema、通常コード |
| Workspace外を読ませない | File Tool、Sandbox、OS |
| Testを必ず実行する | Script、Hook、CI |
| 外部送信前に確認する | Policy、Human Gate |
| 本番DBへ接続させない | Network、Credential、DB Role |
| 完了を判断する | CriterionとEvidenceの照合 |

越えてはならない境界ほど、モデルから独立した場所へ置きます。

---

## 6. 初級者が最初に作る構成

### AIエージェント開発

```text
single agent
+ read-only tool
+ input/output validation
+ max steps
+ structured state
+ two completion criteria
+ evidence report
```

[単一エージェントのチュートリアル](simple-agent-tutorial.md)で作成します。

### コーディングエージェント利用

```text
Task Contract
+ read-only調査
+ Branch
+ workspace内だけの変更
+ verify command
+ git diff
+ Completion Report
```

[コーディングエージェントのチュートリアル](coding-agent-tutorial.md)で実施します。

---

## 7. 上位へ移る境界

次のどれかを扱う場合、この初級構成のまま実行しません。

| 条件 | 移行先 |
|---|---|
| Teamで同じ運用を共有する | [チームハーネス](../intermediate/team-harness.md) |
| CIで非対話実行する | [CI・レビュー・評価](../intermediate/ci-review-and-evaluation.md) |
| Tool PolicyとEvidenceを形式化する | [証拠とポリシー](../intermediate/evidence-and-policy.md) |
| 外部APIの副作用・長時間処理 | [エージェントランタイム](../intermediate/agent-runtime.md) |
| 本番、個人情報、金銭、権限 | [本番高リスクプロファイル](../../profiles/production-high-risk.md) |
| Queue、複数Worker、再開 | [分散実行プロファイル](../../profiles/distributed-execution.md) |

---

## 8. 理解確認

次の問いに自分の言葉で答えられれば、次へ進めます。

1. なぜ「削除しないで」とPromptへ書くだけでは不十分か。
2. Task Contractで「作業」ではなく「成果」をどう表すか。
3. Agentの「Testは通りました」がEvidenceではない理由は何か。
4. 読取りだけのTaskへ書込み権限を与えない理由は何か。
5. 同じErrorが続いたとき、なぜ無限に再試行してはいけないか。
6. 初級者向けガイドと低Riskが同じ意味ではない理由は何か。

次は[単一エージェントのチュートリアル](simple-agent-tutorial.md)へ進んでください。

## 参照

- [中核設計原則](../../core/principles.md)
- [共通用語集](../../core/glossary.md)
- [統合判定モデル](../../core/decision-model.md)
