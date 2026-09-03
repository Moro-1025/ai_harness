---
title: "中級: 証拠・Policy・Human Gateを統合する"
version: "1.2"
core_version: "1.0"
audience: "intermediate"
risk_scope: "medium-to-high"
updated: "2026-09-02"
---

# 証拠・Policy・Human Gateを統合する

## ゴール

多数のControlを列挙するだけでなく、次を一つの判定へ接続します。

```text
Risk
→ Capability
→ Policy Decision
→ Approval
→ Execution
→ Verification Result
→ Evidence Binding
→ Completion Decision
```

---

## 1. Evidenceを「添付File一覧」にしない

弱い形式:

```json
{
  "evidence_refs": [
    "test.log",
    "diff.patch",
    "screenshot.png"
  ]
}
```

この形式だけでは、次が分かりません。

- どのCompletion Criterionを確認したか
- 何を対象にした結果か
- 現在のCommitに対するものか
- 誰・何が検証したか
- Testが成功したか
- 未実行・Blocked・Waivedか
- Writerが書き換えられる場所か
- 期限切れではないか

一つのCriterionごとにVerification Resultを作ります。

```yaml
result_id: VERIFY-123
contract_hash: "sha256:..."
criterion_id: AUTH-001
assertion: "期限切れTokenで401を返す"
status: pass
verifier:
  type: integration-test
  name: auth-integration
  version: "2026.09"
  independent_from_writer: true
subject:
  type: repository
  repository: example/api
  commit: abc1234
evidence_refs:
  - "ci://run/987/artifacts/auth-expired.xml"
verified_at: "2026-09-02T12:00:00Z"
```

---

## 2. Completion Criteriaを設計する

良いCriterion:

- 一つの観測可能なAssertion
- 安定したID
- 必須か任意か
- 検証方法
- 必要Evidence
- 対象
- Failure時の扱い

悪い例:

```text
品質を上げる
使いやすくする
問題を直す
```

良い例:

```text
AUTH-001: 期限切れTokenで401を返す
AUTH-002: 有効TokenのResponse Contractを変更しない
AUTH-003: 認証失敗時にDB書込みが0件である
SCOPE-001: auth/とtests/auth/以外を変更しない
```

### Criterionの粒度

細かすぎると管理Costが増え、粗すぎると何がPassしたか分かりません。

分割の目安:

- 独立したVerifierを持つ
- Failure時の影響が異なる
- Required / Optionalが異なる
- Waiverの可否が異なる
- 対象Versionが異なる

---

## 3. Evidenceの8軸

| 軸 | 確認 |
|---|---|
| 関連性 | Criterionそのものを検証しているか |
| 完全性 | 正常系、失敗系、境界条件を必要な範囲で含むか |
| 独立性 | Writerとは別Process、Context、権限か |
| 改ざん耐性 | Writerが結果・保存先を自由に変更できないか |
| 鮮度 | 現在のCode、Config、Model、Policyに対する結果か |
| 対象固定 | Commit、Artifact Hash、Action Hash、入力へ結び付くか |
| 再現性 | Command、Environment、Datasetを再構築できるか |
| 不確実性 | 未実行、部分結果、観測限界を明記しているか |

Evidenceの「強さ」を一本の順位だけで表しません。

例:

```yaml
evidence_quality:
  relevance: direct
  completeness: boundary-cases-included
  independence: trusted-ci
  tamper_resistance: immutable-artifact
  freshness: current-commit
  subject_binding: commit-and-config
  reproducibility: command-and-image-pinned
  uncertainty: none-known
```

---

## 4. Verification Status

| Status | 意味 | Completionでの扱い |
|---|---|---|
| pass | Criterionを満たすEvidenceがある | 採用候補 |
| fail | Criterionを満たさない | 必須ならReject |
| not_run | 実行していない | 必須ならBlocked |
| blocked | Dependency・Environment不足 | 必須ならBlocked |
| waived | 権限者が理由・期限・Riskを受入れ | Policyに従いHuman判断 |

`not_run`を`pass`へ変換しません。

`waived`には次を要求します。

- WaiveできるCriterionか
- 承認者Role
- 理由
- 期限
- 残存Risk
- Approval Record
- 対象Version
- 事後確認計画

---

## 5. Policy Decision Point

PDPは次を入力とします。

```yaml
subject:
  actor_id: agent-worker-1
  actor_type: agent
  tenant: tenant-a
  roles:
    - coding-worker

task:
  contract_hash: "sha256:..."
  risk_band: high

resource:
  type: repository
  id: example/api
  tenant: tenant-a
  classification: confidential

operation:
  name: create_patch
  side_effect: reversible
  normalized_arguments_hash: "sha256:..."

data_flow:
  derived_classification: confidential
  purpose: bug-fix
  destination:
    type: workspace
    tenant: tenant-a
    maximum_classification: confidential

target:
  expected_version: abc1234

policy:
  id: repository-policy
  version: "4.2"
```

Output:

```yaml
decision: allow
reasons:
  - "Task ScopeとWorkspaceが一致"
obligations:
  - type: logging
    parameters:
      redact_secrets: true
  - type: trusted_verification
    parameters:
      required_before_acceptance: true
```

### 優先順位

```text
独立境界の拒否
> deny
> require_approval
> allow
```

複数のRuleが競合した場合、最も制限の強い判定を採用します。

### Policy Failure

- 低Riskの表示補助など、明示的にFail Open可能なものだけ例外化
- 認可、Credential、個人情報、金銭、外部送信、削除、本番はFail Closed
- Policy Engine障害とPolicy上のDenyを区別して記録
- 使用したPolicy VersionとInput Hashを保存
- 復旧後に必要な再評価を実行

---

## 6. PDPとPEPを分ける

PDPは判断します。PEPは強制します。

| PEP | 強制するもの |
|---|---|
| Tool Gateway | Tool名、Schema、Action Hash |
| File Sandbox | 読取り・書込みPath |
| Network Proxy | Domain、Protocol、Egress |
| Credential Broker | Scope、TTL、Audience |
| Database | Tenant、Row、Operation |
| Git Protection | Push、Merge、Protected Branch |
| Trusted CI | Fixed Commit、Required Check |
| Approval Service | 有効期限、承認者、Consumed状態 |

Promptや自由文のPolicy説明はPDP・PEPの代わりになりません。

---

## 7. Data Flow Policy

### Label伝播

```text
restricted input
+ public input
→ restricted derived output
```

最も高い分類を基本として継承します。Modelが「秘密を除いた」と述べただけで降格しません。

### Sink Policy

```yaml
sink:
  id: public-issue
  external: true
  maximum_classification: public
  allowed_tenants: []
  allowed_purposes:
    - public-support
  requires_dlp: true
  requires_human_approval: true
```

### 独立DLP

外部Sink前に、Modelとは独立して次を検査します。

- Secret pattern
- API Key / Token
- 個人識別子
- Tenant ID
- Internal URL
- Source Code範囲
- Access-controlled Document引用
- Policyで定義した禁止Field

Redaction後は検査結果をEvidenceとして保存し、再分類の根拠にします。

### 能力の分離

最初の防御として、次を同時に有効にしない構成が有効です。

```text
Restricted Dataを読むTool
+
任意の外部Sinkへ書くTool
```

必要な場合は、Read AgentとPublish Agentを分離し、検証済み・Redact済みArtifactだけを境界で渡します。

---

## 8. Human Gateの設計

悪い承認:

```text
この操作を許可しますか？
```

良い承認Packet:

```yaml
operation: "顧客3件へ通知メールを送信"
contract_hash: "sha256:..."
action_hash: "sha256:..."
recipients:
  - customer-101
  - customer-102
  - customer-103
destination: "email"
target_count: 3
irreversible: true
body_diff_ref: "artifact://run/1/message.diff"
policy_decision: "require_approval"
rollback_or_compensation: "送信後の取消し不可。誤送信Runbookを使用"
expires_at: "2026-09-02T13:00:00Z"
```

Trusted UIは正規化引数と実Resourceから生成します。Modelの自由文だけを表示しません。

### 失敗モード

- Model要約だけを見て承認
- 承認疲れ
- 専門性不足
- 作成者・実装者・承認者が同一
- 重要差分が長文へ埋没
- 承認後に引数・対象が変わる
- 古いApprovalのReplay
- 承認画面と実行Actionの不一致

### 対策

- Action Hash
- 対象Version
- 承認TTL
- One-time consumption
- 実行直前再評価
- Role-based approval
- 高Risk時のSeparation of Duties
- Approval率、差戻し率、上書き率、判断時間の測定
- Break-glassの事後Review

---

## 9. Reviewer Assurance

独立性の軸:

| 軸 | 弱い | 強い |
|---|---|---|
| Context | Writerと同じ | Fresh |
| Workspace | Dirty | Clean Checkout |
| Permission | Write可能 | read-only |
| Evaluator | 同じModel・Prompt | 別Model / 別方式 |
| Execution | Writer生成Logだけ | Trusted CI再実行 |
| Expertise | 一般 | Domain / Security専門 |
| Evidence Source | Writer要約 | Contract、Diff、Source、Artifact |

Profile例:

```yaml
review_profile: high-risk-code-change
context: fresh
workspace: clean_checkout
permissions: read_only
evaluator:
  type: different-model
execution:
  trusted_ci: required
expertise:
  - security
human_review: required
```

別ModelはTestの代わりではなく、TestはRequirement判断の代わりではありません。

---

## 10. Completion Decision

Acceptance ServiceまたはTrusted CIで次を計算します。

```text
required_criteria_complete
evidence_binding_check
scope_check
policy_check
critical_unresolved_errors
uncertain_actions
residual_risks
```

`accepted`条件:

```text
required_criteria_complete == true
AND evidence_binding_check == pass
AND scope_check == pass
AND policy_check == pass
AND critical_unresolved_errors == 0
AND uncertain_actions is empty
```

Schemaで表現できない意味検証もあります。

- Verification参照先が本当に存在する
- Hashが正しいCanonical Inputから作られた
- Required Approvalsの人数・Roleが満たされる
- Evidence Artifactが改ざんされていない
- Criterion IDがActive Contractに存在する
- Approval時と実行時の対象Versionが同じ

これらはRuntime、Policy Service、Trusted CIで検証します。

---

## 11. 実装順序

1. Completion CriterionへIDを付ける
2. CriterionごとのVerification Resultを保存する
3. Subject Commit / Versionを固定する
4. RiskとCapabilityを分離する
5. Policy Decisionの共通形式を作る
6. `deny > require_approval > allow`を統一する
7. Approval PacketとAction Hashを導入する
8. Data ClassificationとSink上限を追加する
9. Independent DLPを追加する
10. Completion Decisionを機械化する

最初から中央Policy Serviceを構築する必要はありません。Local Tool Gatewayでも、同じInput / Output契約を使えば段階的に移行できます。

---

## 12. 完了条件

- Criterionへ安定したIDがある
- EvidenceをCriterionとSubject Versionへ結び付ける
- Pass / Fail / Not Run / Blocked / Waivedを分ける
- Risk BandとCapability Profileを分ける
- Policyの入力、出力、Version、優先順位を定義する
- Policy Failure時のFail Open / Closedを明示する
- PDPとPEPを分ける
- Classificationを派生出力へ伝播する
- Sinkごとの上限を持つ
- Modelから独立したDLPを外部送信前に置く
- Human GateをAction Hashと対象Versionへ結び付ける
- Reviewer Assuranceを多軸で記録する
- Completion DecisionをAgent外で検証する

## 参照

- [統合判定モデル](../../core/decision-model.md)
- [Policy Decision Schema](../../core/schemas/policy-decision.schema.json)
- [Approval Record Schema](../../core/schemas/approval-record.schema.json)
- [Verification Result Schema](../../core/schemas/verification-result.schema.json)
- [Completion Decision Schema](../../core/schemas/completion-decision.schema.json)
- [本番高リスクプロファイル](../../profiles/production-high-risk.md)
