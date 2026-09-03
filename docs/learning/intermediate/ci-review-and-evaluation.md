---
title: "中級: CI・独立レビュー・統計的評価"
version: "1.2"
core_version: "1.0"
audience: "intermediate"
risk_scope: "medium-to-high"
updated: "2026-09-02"
---

# CI・独立レビュー・統計的評価

## ゴール

Localで動いたAgentを、そのまま自動化しません。

```text
固定Task
→ 隔離された実行
→ Machine-readable Output
→ Deterministic CI
→ Agent Review
→ Human Review
→ Evidence付きRelease Decision
```

さらに、Harness変更を少数の成功例ではなく、Version付きDatasetと不確実性を含む比較で判断します。

---

## 1. 非対話実行は別の製品面である

対話Sessionでは、曖昧さを人間へ質問できます。CIでは、質問できないまま処理が続く可能性があります。

CI開始前に固定するもの:

- Task Contract
- Starting Commit
- Capability Profile
- Network
- Credential
- Allowed Tool
- Timeout / Budget
- Required Integration
- Output Schema
- Evidence Location
- Stop / Failure Policy

不明な状態で継続するより、明示的にFail Closedします。

### Startup Failure

次を黙って省略しません。

- 必須Policyが読み込めない
- Output Schemaが利用できない
- Required MCP / Toolが起動しない
- Starting Commitを固定できない
- Trusted Verification Scriptが存在しない
- Artifact Storeへ保存できない
- Secret Brokerが利用できない

---

## 2. CI Environment

推奨:

- Isolated Runner
- Clean Checkout
- Fixed Commit
- Dedicated Workspace
- Network Allowlist
- Short-lived Credential
- SecretをAgent Processだけへ限定
- Time / CPU / Memory上限
- Run後の破棄
- Artifact Retention
- WriterがCI定義を同じ変更で無審査に変更できない保護

高Riskでは、HarnessやCI Workflowの変更自体を別OwnerがReviewします。

### Credential

避ける構成:

```text
Repository-controlled build / tests
+
長期Credential
+
同じJob全体のEnvironment
```

推奨:

```text
Setup Job
→ Clean Artifact
→ Agent Job（必要最小Credential）
→ Credential破棄
→ Verification Job（原則write不可）
```

---

## 3. Deterministic CIとAgent Reviewを分ける

```text
Deterministic CI
+ Agent Review
+ Human Review
```

### Deterministic CI

- Build
- Format
- Lint
- Type Check
- Unit / Integration / Contract Test
- Schema
- Secret Scan
- License
- Generated File一致
- Scope / Forbidden File
- Policy
- Artifact Hash

### Agent Review

- Logicの抜け
- Requirementとの不一致
- Missing Test
- Architecture Drift
- Security Smell
- Documentation mismatch
- 複雑性・保守性

### Human Review

- Requirement解釈
- Product価値
- UX
- Architecture Trade-off
- Risk Acceptance
- 法務・契約
- 本番移行

Agent Reviewだけを唯一のMerge Gateにしません。逆に、Deterministic CIだけで意味的正しさが保証されるとも考えません。

---

## 4. WriterとVerifierを分離する

推奨Pipeline:

```text
Writer Job
  └─ Patch + Agent Evidence

Fixed Commit
  ↓

Trusted Verification Job
  └─ Deterministic Evidence

Fresh Reviewer Job
  └─ Findings

Human Acceptance
  └─ Completion Decision
```

### Writer Job

- workspace-write
- Network原則off
- Patch作成
- Project verifyによる自己検証
- BranchへCommitまたはPatch Artifact

### Verification Job

- Clean Checkout
- Fixed Commit
- 原則read-only
- Trusted Script
- Writerが変更できないArtifact Store
- CriterionごとのVerification Result

### Reviewer Job

- Fresh Context
- read-only
- Task Contract、Diff、Source、Trusted Evidenceを読む
- Findingだけを返す
- 修正しない
- Output Schemaに従う

---

## 5. Machine-readable Output

自然言語Summaryだけで後段を制御しません。

Review Result例:

```json
{
  "verdict": "needs_changes",
  "subject_commit": "abc1234",
  "findings": [
    {
      "severity": "high",
      "criterion_id": "AUTH-002",
      "path": "src/auth/refresh.py",
      "line": 88,
      "title": "有効Tokenも401へ変換される",
      "evidence": "catch節がToken状態を区別していない",
      "recommendation": "expired errorだけをmappingする"
    }
  ],
  "missing_verification": [
    "有効TokenのIntegration Test"
  ]
}
```

Schemaは形を保証します。内容の真偽はSource、Test、Policyで別途確認します。

---

## 6. Artifactを対象Versionへ結び付ける

保存候補:

```text
task-contract.json
contract-hash.txt
starting-commit.txt
ending-commit.txt
config-summary.json
agent-events.jsonl
patch.diff
verification-results/
review-result.json
policy-decisions/
approval-records/
completion-decision.json
```

各Artifactへ次を付けます。

- Run ID
- Task ID
- Contract Hash
- Starting / Ending Commit
- Harness Version
- Model / Product Version
- Policy Version
- Environment Image / Dependency Lock
- Created At
- Classification
- Retention
- Hash / Signature

---

## 7. Evalの4層

### Component

- Tool選択
- 引数精度
- Retrieval
- Structured Output
- Memory検索
- Policy Decision

### Trajectory

- 不要Step
- 同一Action反復
- Plan逸脱
- Budget超過
- Human Escalation時点
- Subagent Coordination

### Outcome

- Task Completion
- Criterion達成
- Scope遵守
- 副作用の正しさ
- Evidence完全性
- Reviewer Acceptance

### Operational

- Latency
- Cost
- Retry
- Human Intervention
- Rework
- Policy Violation
- Rollback
- Incident
- Cost per Accepted Change

---

## 8. Eval DatasetをVersion管理する

Case:

```yaml
case_id: auth-expired-token
dataset_version: "2026.09"
task_type: bug-fix
risk_band: medium
repository_size: small
starting_commit: abc1234
task_contract_ref: cases/auth-expired-token/task.json
expected:
  criteria:
    AUTH-001: pass
    AUTH-002: pass
    AUTH-003: pass
  forbidden_changes:
    - migrations/**
required_evidence:
  - integration-test
  - diff
  - completion-decision
reviewer_rubric_version: "2.1"
```

Datasetに含めるもの:

- 正常系
- 境界値
- Failure
- Permission不足
- Missing Data
- 古いMemory
- Tool Timeout
- Prompt Injection
- Human Gate
- StopすべきCase
- Uncertainな外部結果
- Scope変更が必要なCase

本番Incidentを匿名化・最小化し、Regression Caseへ追加します。

---

## 9. Train、Development、Holdoutを分ける

- **Development Set**: Harness開発中に繰り返し見る
- **Holdout Set**: Release判断のため、日常のPrompt調整へ直接使わない
- **Incident Set**: 既知の本番失敗を必ず再検証する
- **Adversarial Set**: Injection、権限、Data流出、Policy回避
- **Shadow / Production Sample**: 本番分布との差を見る

同じCaseと期待回答をModel Promptへ含めると、一般化ではなく暗記を測る可能性があります。

---

## 10. 層別評価

全Task平均だけでは問題が隠れます。

最低限の層:

- Task Type
- Risk Band
- Repository規模
- Language / Framework
- Tool構成
- Model / Harness Version
- Interactive / CI
- New / Existing Code
- Data Classification

例:

```text
全体成功率は改善
しかし
high-risk migrationだけ低下
```

この場合、全体平均が良くてもReleaseしない判断があり得ます。

---

## 11. 最小Sample数と信頼区間

成功率`92%`だけでは、Sample数が10件か1000件か分かりません。

Release Gateへ次を含めます。

- `n`
- 成功数 / Failure数
- 信頼区間
- Baselineとの差
- 対応のあるCase比較
- Task層別結果
- Severity-weighted Failure
- Flaky率

例:

```yaml
metric:
  name: task_success
  successes: 184
  total: 200
  estimate: 0.92
  confidence_interval_95:
    lower: 0.87
    upper: 0.95
```

閾値は点推定だけでなく、保守的な側の信頼限界で判断します。

### 対応比較

同じCaseを変更前後で実行し、次を分類します。

- both pass
- before pass / after fail
- before fail / after pass
- both fail

単純な独立平均より、どのCaseが回帰・改善したかを確認しやすくなります。

---

## 12. Safety Failureが0件でも安全とは限らない

100件の評価で違反が0件でも、真の違反率が0とは証明できません。

Release時は次を見る必要があります。

- Sample数
- Safety指標の上側信頼限界
- FailureのSeverity
- Adversarial Coverage
- 本番Exposure量
- 検出能力
- Drift

重大なSafety Failureは、一般Task成功率で相殺しません。

```yaml
safety_gate:
  critical_violations: 0
  high_violations: 0
  upper_confidence_bound_below: organization_defined_limit
  adversarial_cases_complete: true
```

具体的な許容値は、対象業務、法務、Risk Appetite、Exposureから決めます。

---

## 13. Flaky Case

同じ構成で結果が揺れるCaseを、都合よく除外しません。

記録:

- Run回数
- Pass / Fail分布
- Failure理由
- Model Non-determinism
- External Dependency
- Test Flakiness
- Timeout
- Environment差

扱い:

- Deterministic TestのFlakyは修正する
- Modelの変動は複数Runと分布で測る
- External ServiceはStub / ReplayとLiveを分ける
- Flaky CaseをRelease Gateから除外する場合、理由と影響を記録する

---

## 14. Severity-weighted Failure

すべてのFailureを同じ1件として扱いません。

例:

| Severity | 例 | 扱い |
|---|---|---|
| critical | Data流出、二重決済、権限昇格 | 即Block |
| high | 本番破壊、重大Security欠陥 | Release Block |
| medium | Task Failure、Scope外変更 | 重み付き評価 |
| low | 冗長Step、軽微な文体 | 改善候補 |

Severity Weightは補助指標です。Critical Failureを平均Scoreで相殺しません。

---

## 15. Contaminationを検査する

Signal:

- Eval Case名や期待OutputがPrompt・Memoryへ混入
- HoldoutのSourceがTraining例として利用
- Reviewer Rubricを生成側が直接最適化
- Case固有のHard-coded Rule
- Test File名から正解を推測
- Incident Caseの答えを長期Memoryへ保存

対策:

- Dataset Accessの分離
- Holdout Owner
- Case IDを推論へ不要に露出しない
- Test fixtureの変形
- 新規Caseの定期追加
- Prompt / Memory / RetrievalのAudit
- Human作成のBlind Review

---

## 16. Drift

比較する分布:

- Task Type
- Input長
- Repository規模
- Tool使用率
- Risk Band
- Language / Framework
- Human介入率
- Failure Reason
- Data Classification
- Latency / Cost

本番分布がDatasetから離れた場合:

1. 代表Sampleを匿名化
2. Datasetへ追加
3. 過去Versionで再Baseline
4. Release Gateを再評価
5. 必要ならProfileまたはCapabilityを制限

---

## 17. Harness Release Gate

例:

```yaml
release_gate:
  dataset_version: "2026.09"
  minimum_total_cases: 200
  required_strata:
    - bug-fix
    - review
    - documentation
    - security
  no_regression_on_incident_set: true
  critical_safety_violations: 0
  high_safety_violations: 0
  maximum_scope_violation_upper_bound: organization_defined
  task_success_lower_bound: organization_defined
  cost_per_accepted_change_regression: organization_defined
  human_review_minutes_regression: organization_defined
  flaky_case_policy: documented
  contamination_review: pass
```

値は説明用ではなく、組織の実測・SLO・Risk Appetiteから決めます。最初から200件を必須にするのではなく、初期段階はCase数と不確実性を明記して人間が判断し、Datasetを段階的に増やします。

---

## 18. 完了条件

- CIでTask、Commit、Profile、Budgetを固定する
- Required Integration不足時にFail Closedする
- CredentialをRepository-controlled codeから分離する
- Writer、Trusted Verification、Reviewerを分ける
- Machine-readable Output Schemaがある
- ArtifactをContract HashとCommitへ結び付ける
- Component / Trajectory / Outcome / Operationalを分ける
- Dataset VersionとHoldoutを持つ
- Task種別・Risk・規模で層別する
- Sample数と信頼区間を報告する
- Safety 0件を安全の証明にしない
- Flaky、Contamination、Driftを管理する
- Critical Failureを平均Scoreで相殺しない
- Harness変更前後で同一Caseを比較する

## 参照

- [チームハーネス](team-harness.md)
- [証拠とポリシー](evidence-and-policy.md)
- [チームCIプロファイル](../../profiles/team-ci.md)
- [Verification Result Schema](../../core/schemas/verification-result.schema.json)
- [Completion Decision Schema](../../core/schemas/completion-decision.schema.json)
