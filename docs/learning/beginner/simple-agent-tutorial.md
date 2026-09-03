---
title: "初級: 読取り専用の単一エージェントを作る"
version: "1.2"
core_version: "1.0"
audience: "beginner"
risk_scope: "low"
updated: "2026-09-02"
---

# 読取り専用の単一エージェントを作る

## 目的

このチュートリアルでは、次のTaskを行う小さなハーネスを作ります。

> `workspace/README.md`を読み、Project名を根拠付きで返す。

最初はModel部分を決定論的なSimulatorにします。これにより、API Key、Network、Provider差異なしで、Agent Loop、Tool検証、Budget、State、Evidence、Completionを確認できます。最後にModel Adapterへ置き換える場所を示します。

### 対象外

- File書込み
- Network
- Shell
- Credential
- 外部副作用
- Multi-Agent
- 長期Memory

---

## 1. 作業Directoryを作る

```text
simple-agent/
├─ simple_agent.py
└─ workspace/
   └─ README.md
```

`workspace/README.md`:

```markdown
# Sample Harness Project

安全な単一エージェントの練習用Projectです。
```

---

## 2. Task Contractを定義する

今回の契約は次です。

```yaml
task_id: tutorial-read-project-name
objective: "READMEからProject名を確認する"
scope:
  read:
    - "workspace/**"
  write: []
  forbidden:
    - "../**"
completion_criteria:
  - criterion_id: PROJECT-001
    assertion: "Project名を回答する"
    required: true
  - criterion_id: PROJECT-002
    assertion: "根拠となるFileを示す"
    required: true
capability_profile:
  filesystem: read
  network: denied
  external_side_effects: denied
  credential_access: none
execution_budget:
  max_steps: 5
  same_action_limit: 1
stop_conditions:
  - scope_boundary_exceeded
  - budget_exceeded
  - repeated_action
  - source_missing
```

本番実装では[Task Contract Schema](../../core/schemas/task-contract.schema.json)へ合わせます。このチュートリアルでは概念を見やすくするため、必要最小限のFieldだけをCodeへ入れます。

---

## 3. 実装する

`simple_agent.py`:

```python
from __future__ import annotations

from dataclasses import dataclass, field
from pathlib import Path
from typing import Any, Literal


WORKSPACE = Path(__file__).parent.joinpath("workspace").resolve()
MAX_STEPS = 5

ActionName = Literal["list_files", "read_file", "finish"]


@dataclass(frozen=True)
class Action:
    name: ActionName
    arguments: dict[str, Any]


@dataclass
class State:
    step: int = 0
    files: list[str] = field(default_factory=list)
    content: str | None = None
    evidence: list[str] = field(default_factory=list)
    action_fingerprints: set[str] = field(default_factory=set)


def resolve_inside_workspace(relative_path: str) -> Path:
    """Path traversalとWorkspace外参照を拒否する。"""
    candidate = WORKSPACE.joinpath(relative_path).resolve()

    try:
        candidate.relative_to(WORKSPACE)
    except ValueError as exc:
        raise PermissionError(f"scope_boundary_exceeded: {relative_path}") from exc

    return candidate


def list_files() -> dict[str, Any]:
    files = [
        path.relative_to(WORKSPACE).as_posix()
        for path in WORKSPACE.rglob("*")
        if path.is_file()
    ]
    return {"ok": True, "files": sorted(files)}


def read_file(path: str) -> dict[str, Any]:
    target = resolve_inside_workspace(path)

    if not target.is_file():
        return {
            "ok": False,
            "error_code": "SOURCE_MISSING",
            "message": f"File not found: {path}",
        }

    return {
        "ok": True,
        "path": path,
        "content": target.read_text(encoding="utf-8"),
    }


def simulated_model(state: State) -> Action:
    """
    Harnessを学ぶための決定論的Simulator。
    後でLLM Adapterへ置き換える。
    """
    if not state.files:
        return Action("list_files", {})

    if state.content is None:
        if "README.md" not in state.files:
            return Action(
                "finish",
                {"status": "blocked", "reason": "README.mdが存在しない"},
            )
        return Action("read_file", {"path": "README.md"})

    first_line = state.content.splitlines()[0].removeprefix("# ").strip()
    return Action(
        "finish",
        {
            "status": "completed",
            "project_name": first_line,
            "source": "workspace/README.md:1",
        },
    )


def validate_action(action: Action) -> None:
    """Modelの提案を実行前に決定論的に検証する。"""
    allowed: set[str] = {"list_files", "read_file", "finish"}

    if action.name not in allowed:
        raise PermissionError(f"tool_not_allowed: {action.name}")

    if action.name == "read_file":
        path = action.arguments.get("path")
        if not isinstance(path, str) or not path:
            raise ValueError("invalid_tool_arguments: path is required")
        resolve_inside_workspace(path)

    if action.name in {"list_files", "finish"} and not isinstance(
        action.arguments, dict
    ):
        raise ValueError("invalid_tool_arguments")


def action_fingerprint(action: Action) -> str:
    return f"{action.name}:{sorted(action.arguments.items())!r}"


def state_snapshot(state: State) -> dict[str, Any]:
    """JSONへ保存できる形へ正規化する。"""
    return {
        "step": state.step,
        "files": state.files,
        "content": state.content,
        "evidence": state.evidence,
        "action_fingerprints": sorted(state.action_fingerprints),
    }


def verify_completion(
    state: State, result: dict[str, Any]
) -> dict[str, Any]:
    criteria = {
        "PROJECT-001": {
            "status": "pass" if result.get("project_name") else "fail",
            "evidence": ["completion.project_name"],
        },
        "PROJECT-002": {
            "status": "pass" if result.get("source") else "fail",
            "evidence": state.evidence + ["completion.source"],
        },
    }

    accepted = all(item["status"] == "pass" for item in criteria.values())
    return {
        "status": "accepted" if accepted else "rejected",
        "criteria": criteria,
        "result": result,
    }


def run() -> dict[str, Any]:
    state = State()

    while True:
        if state.step >= MAX_STEPS:
            return {
                "status": "blocked",
                "stop_reason": "budget_exceeded",
                "state": state_snapshot(state),
            }

        action = simulated_model(state)
        validate_action(action)

        fingerprint = action_fingerprint(action)
        if fingerprint in state.action_fingerprints:
            return {
                "status": "blocked",
                "stop_reason": "repeated_action",
                "state": state_snapshot(state),
            }

        state.action_fingerprints.add(fingerprint)
        state.step += 1

        if action.name == "list_files":
            result = list_files()
            if not result["ok"]:
                return {
                    "status": "blocked",
                    "stop_reason": result["error_code"],
                }
            state.files = result["files"]
            state.evidence.append("workspace file listing")
            continue

        if action.name == "read_file":
            result = read_file(action.arguments["path"])
            if not result["ok"]:
                return {
                    "status": "blocked",
                    "stop_reason": result["error_code"],
                    "details": result["message"],
                }
            state.content = result["content"]
            state.evidence.append(
                f"workspace/{result['path']}:1"
            )
            continue

        if action.name == "finish":
            if action.arguments.get("status") != "completed":
                return {
                    "status": "blocked",
                    "stop_reason": action.arguments.get(
                        "reason", "model_stopped"
                    ),
                }
            return verify_completion(state, action.arguments)


if __name__ == "__main__":
    import json

    print(json.dumps(run(), ensure_ascii=False, indent=2))
```

---

## 4. 実行する

```bash
python simple_agent.py
```

期待する結果:

```json
{
  "status": "accepted",
  "criteria": {
    "PROJECT-001": {
      "status": "pass",
      "evidence": [
        "completion.project_name"
      ]
    },
    "PROJECT-002": {
      "status": "pass",
      "evidence": [
        "workspace file listing",
        "workspace/README.md:1",
        "completion.source"
      ]
    }
  },
  "result": {
    "status": "completed",
    "project_name": "Sample Harness Project",
    "source": "workspace/README.md:1"
  }
}
```

重要なのは回答の自然さではなく、次が成立していることです。

- Model候補を実行前に検証する
- Workspace外へ出られない
- 書込みToolが存在しない
- Step上限がある
- 同じActionの反復で止まる
- Criterionごとに結果とEvidenceがある
- Agentの`completed`だけではAcceptanceにならない

---

## 5. 失敗を試す

### Scope外参照

`simulated_model`のPathを次へ変えます。

```python
return Action("read_file", {"path": "../secret.txt"})
```

`scope_boundary_exceeded`で停止すれば成功です。Promptで禁止したのではなく、`resolve_inside_workspace`が拒否しています。

### Sourceなし

`workspace/README.md`を一時的に別名へ変更します。

結果は`README.mdが存在しない`として`blocked`になります。存在しない情報を推測して完了してはいけません。

### 予算不足

`MAX_STEPS = 1`へ変更します。

File一覧を取得した後、`budget_exceeded`で止まります。未完了を成功扱いしないことを確認します。

### 同一Action

`simulated_model`を常に`list_files`を返すようにします。

2回目の実行前に`repeated_action`で停止します。

---

## 6. LLMへ置き換える場所

置き換えるのは`simulated_model`だけです。

```python
def model_adapter(state: State) -> Action:
    response = your_model_client.generate(
        system=(
            "You may propose only list_files, read_file, or finish. "
            "Do not assume tool results."
        ),
        input={
            "step": state.step,
            "files": state.files,
            "content": state.content,
        },
        output_schema={
            "name": "list_files | read_file | finish",
            "arguments": "object",
        },
    )
    return Action(
        name=response["name"],
        arguments=response["arguments"],
    )
```

次は置き換えません。

- `validate_action`
- `resolve_inside_workspace`
- `MAX_STEPS`
- 反復検出
- Tool実装
- `verify_completion`
- Evidence記録

モデルが変わっても、安全境界と完了判定は残ります。

---

## 7. 次に追加するもの

一度に複雑化しません。実際に失敗が確認された場合だけ追加します。

| 観測した失敗 | 追加候補 |
|---|---|
| Tool引数の形が崩れる | JSON Schema |
| 大きなFileを読みすぎる | Read上限、Chunking |
| 古い情報を使う | Source時刻、Freshness |
| 長いTaskで状態を失う | Checkpoint Store |
| Tool Timeout | Error分類と制限付きRetry |
| 外部副作用が必要 | Idempotency、Policy、Human Gate |
| 複数Workerが必要 | [分散実行プロファイル](../../profiles/distributed-execution.md) |

---

## 8. 完了条件

このチュートリアルは、次を確認できれば完了です。

- 正常系で`accepted`になる
- `../secret.txt`が拒否される
- READMEなしで推測せず`blocked`になる
- Budget超過で停止する
- 同一Action反復で停止する
- CriterionごとのEvidenceを説明できる
- LLM AdapterとHarness責任を分離して説明できる

次は[コーディングエージェントのチュートリアル](coding-agent-tutorial.md)へ進んでください。

## 参照

- [ハーネスの基礎](harness-basics.md)
- [中核設計原則](../../core/principles.md)
- [統合判定モデル](../../core/decision-model.md)
- [エージェントランタイム](../intermediate/agent-runtime.md)
