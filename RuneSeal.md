# RuneSeal

（Rust 2021 / **MSRV 1.80** でビルド保証）

---

## 0. ゴール

- Pull Request を自動評価し、**Approve / Changes Requested / Reject / Merge** を機械判定。
    
- 人間は最終署名 1 クリックのみ。決定は JSON で**不可逆保存**。
    
- 判定結果を **`repository_dispatch`** で次段（RuneTrial）へ通知。([GitHub Docs](https://docs.github.com/en/enterprise-server%403.17/actions/writing-workflows/choosing-when-your-workflow-runs/triggering-a-workflow?utm_source=chatgpt.com "Triggering a workflow - GitHub Enterprise Server 3.17 Docs"), [TryAPIs](https://tryapis.com/github/api/repos-create-dispatch-event/?utm_source=chatgpt.com "GitHub API | Create a repository dispatch event - tryapis.com"))
    

---

## 1. 入出力

|種別|形式|概要|
|---|---|---|
|入力|GitHub PR|`pull_request` / `workflow_run` / `pull_request_review` / `runeweave_done`（dispatch）を監視。([GitHub Docs](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows?from=20421&utm_source=chatgpt.com "Events that trigger workflows - GitHub Docs"), [GitHub](https://github.com/orgs/community/discussions/25372?utm_source=chatgpt.com "Feature Request \|trigger action on \"Pull Request Approved\""))|
||`.runeseal/prefs.yml`|好み（**Hard-fail しない**）|
||`.runeseal/policy.yml`|**必須**（Fail 時 auto-merge 不可）|
|出力|`.runeseal/report.json`|`{ verdict, reasons[], signature, plan: { checks[], reviews[], diff_metrics }, hash }`|
|出力|GitHub 操作|approve / changes_requested / merge / close（REST/GraphQL 経由）。([GitHub Docs](https://docs.github.com/en/enterprise-cloud%40latest/rest/pulls?utm_source=chatgpt.com "REST API endpoints for pull requests - GitHub Docs"))|
|出力|`repository_dispatch`|`{ event_type: "rune_seal_{granted|

### CLI

```bash
runeseal evaluate-merge \
  --pr 123 \
  --prefs .runeseal/prefs.yml \
  --policy .runeseal/policy.yml \
  --out .runeseal/report.json \
  --auto-merge          # 署名付き rebase-merge
```

|Exit|意味|
|---|---|
|0|approved|
|2|changes requested|
|3|rejected|

---

## 2. 評価フロー

|順|ゲート|Fail 時|
|---|---|---|
|1|**Security Hard-Fail**（秘密漏えい / 禁止ライセンス / 危険パターン）|reject + close|
|2|**必須 Checks**（`policy.required_checks`）完了＆success|changes|
|3|**Quality Gates**（coverage_min / bundle_max 等）|changes|
|4|**Prefs** 逸脱（命名・スタイル等）|コメント + changes|
|5|**Manual Seal（署名）**|approve|
|6|**署名 Merge**（rebase/squash/merge）|GitHub API 実行。レビュー状態は **APPROVE / REQUEST_CHANGES / COMMENT** を使用。([GitHub](https://github.com/octokit/plugin-rest-endpoint-methods.js/blob/master/docs/pulls/merge.md?utm_source=chatgpt.com "plugin-rest-endpoint-methods.js/docs/pulls/merge.md at main - GitHub"), [GitHub Docs](https://docs.github.com/en/enterprise-cloud%40latest/rest/pulls/reviews?utm_source=chatgpt.com "REST API endpoints for pull request reviews - GitHub Docs"))|

- レビューの状態は GitHub が提供する 3 区分を厳密に反映。([GitHub Docs](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/approving-a-pull-request-with-required-reviews?apiVersion=2022-11-28&utm_source=chatgpt.com "Approving a pull request with required reviews - GitHub Docs"))
    
- Merge 方式は PR 設定に従い **merge / squash / rebase** を選択。([GitHub Docs](https://docs.github.com/en/enterprise-cloud%40latest/rest/pulls?utm_source=chatgpt.com "REST API endpoints for pull requests - GitHub Docs"))
    

---

## 3. GitHub Actions テンプレ

```yaml
name: RuneSeal
on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]                # CI 成功後に評価へ。:contentReference[oaicite:7]{index=7}
  pull_request:
    types: [opened, synchronize, reopened]
  pull_request_review:
    types: [submitted]                # 承認/差戻タイミングも監視。:contentReference[oaicite:8]{index=8}
  repository_dispatch:
    types: [runeweave_done]           # 生成完了トリガ。:contentReference[oaicite:9]{index=9}

jobs:
  seal:
    runs-on: ubuntu-24.04
    permissions:
      contents: write                 # rebase-merge 等に必要。:contentReference[oaicite:10]{index=10}
      pull-requests: write            # Review/merge 操作用。:contentReference[oaicite:11]{index=11}
      checks: read
      statuses: read
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - name: Install runeseal
        run: cargo install --git https://github.com/runetools/runeseal
      - name: Evaluate & Merge
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          runeseal evaluate-merge \
            --pr ${{ github.event.pull_request.number || github.event.workflow_run.pull_requests[0].number || github.event.client_payload.pr }} \
            --prefs .runeseal/prefs.yml \
            --policy .runeseal/policy.yml \
            --out .runeseal/report.json \
            --auto-merge
```

> **注意**: フォーク PR では `GITHUB_TOKEN` が既定で **read-only** となり、書き込み操作（approve/merge）は不可。必要に応じて「**Send write tokens to workflows from pull requests**」設定や手動承認で回避。([GitHub Docs](https://docs.github.com/en/enterprise-cloud%40latest/actions/reference/github_token-reference?utm_source=chatgpt.com "GITHUB_TOKEN reference - GitHub Enterprise Cloud Docs"))

---

## 4. ポリシー DSL

### `.runeseal/prefs.yml`（例：**Hard-fail しない**好み）

```yaml
version: 1
style:
  frontend:
    dark_mode: true
review:
  nitpick_level: low
comment:
  locale: ja-JP
```

### `.runeseal/policy.yml`（**必須**）

```yaml
version: 1
required_checks: ["ci/build", "ci/test", "security/sast"]   # Checks API/Statuses を集約。:contentReference[oaicite:13]{index=13}
quality_gates:
  coverage_min: 0.75
  bundle_max_kb: 300
security:
  blocked_licenses: ["AGPL-3.0"]
  deny_patterns:
    - "eval\\("
    - "curl\\s+.+\\|\\s*bash"
merge:
  strategy: rebase
  require_signed_merge: true      # 署名済みマージを強制（Verified バッジ）。:contentReference[oaicite:14]{index=14}
dispatch:
  on_success: "rune_seal_granted"
  on_reject:  "rune_seal_rejected"
```

---

## 5. 受け入れ基準（DoD）

1. **Security 違反 PR** → 自動 close + `rune_seal_rejected` を dispatch。([TryAPIs](https://tryapis.com/github/api/repos-create-dispatch-event/?utm_source=chatgpt.com "GitHub API | Create a repository dispatch event - tryapis.com"))
    
2. **必須 Checks 未達** → `changes_requested` を付与（レビュー API）。([GitHub Docs](https://docs.github.com/en/enterprise-cloud%40latest/rest/pulls/reviews?utm_source=chatgpt.com "REST API endpoints for pull request reviews - GitHub Docs"))
    
3. **署名マージ** 実施時、PR のコミットに **“Verified”** バッジが付与。([GitHub Docs](https://docs.github.com/enterprise-cloud%40latest//articles/signing-commits-with-gpg?utm_source=chatgpt.com "Managing commit signature verification - GitHub Docs"))
    
4. `repository_dispatch` が受信側ワークフローを起動（次段 RuneTrial）。([GitHub Docs](https://docs.github.com/en/enterprise-server%403.17/actions/writing-workflows/choosing-when-your-workflow-runs/triggering-a-workflow?utm_source=chatgpt.com "Triggering a workflow - GitHub Enterprise Server 3.17 Docs"))
    
5. `runeseal audit` で過去の決定（`report.json`）を PlanetScale/R2 に保存・参照可能。
    

---

## 6. 実装レイアウト

```
/src
  main.rs        # clap CLI
  evaluate.rs    # 各ゲート実装（Checks/Reviews/Prefs）
  github.rs      # PR/Review/Merge 操作（REST/GraphQL）
  policy.rs      # prefs + policy DSL
  signer.rs      # GPG/SSH 署名・検証
  audit.rs       # PlanetScale + R2 保存
```

### 依存（固定ピン）

|crate|ver|用途|
|---|---|---|
|`clap`|4.5|CLI|
|`octocrab`|**0.44**|GitHub API（Pulls/Reviews/Checks/GraphQL）。([Docs.rs](https://docs.rs/octocrab/latest/octocrab/ "octocrab - Rust"))|
|`serde` / `serde_yaml` / `serde_json`|1 / 0.9 / 1|I/O|
|`schemars`|0.8|Schema 検証|
|`git2`|**0.20**|署名コミット作成・rebase/squash 補助。([Docs.rs](https://docs.rs/crate/git2/latest?utm_source=chatgpt.com "git2 0.20.2 - Docs.rs"))|
|`ring`|0.17|署名/ハッシュ（Verified バッジ対象の署名生成補助）。([Docs.rs](https://docs.rs/ring/latest/ring/?utm_source=chatgpt.com "ring - Rust - Docs.rs"))|

> 備考：PR の **approve / request_changes / merge** は GitHub API（Pulls & Reviews エンドポイント）で実施。([GitHub Docs](https://docs.github.com/en/enterprise-cloud%40latest/rest/pulls?utm_source=chatgpt.com "REST API endpoints for pull requests - GitHub Docs"))

---

## 7. 判定アルゴリズム（概要）

```text
inputs: PR metadata, diff stats, CI checks, code scanning alerts, prefs, policy
S = SecurityGate(PR, policy.security)            # Hard-fail
C = ChecksGate(PR, policy.required_checks)       # 必須チェック収束
Q = QualityGate(metrics, policy.quality_gates)   # しきい値
P = PrefsGate(prefs)                             
verdict = if S=fail -> REJECT
          else if C=fail or Q=fail -> CHANGES_REQUESTED
          else -> APPROVE
merge = policy.merge.strategy if verdict=APPROVE && manual_seal_signed
```

- レビュー操作は **APPROVE / REQUEST_CHANGES / COMMENT** の 3 値。([GitHub Docs](https://docs.github.com/en/enterprise-cloud%40latest/rest/pulls/reviews?utm_source=chatgpt.com "REST API endpoints for pull request reviews - GitHub Docs"))
    
- Merge は PR 設定により **merge / squash / rebase** を使い分け。([GitHub Docs](https://docs.github.com/en/enterprise-cloud%40latest/rest/pulls?utm_source=chatgpt.com "REST API endpoints for pull requests - GitHub Docs"))
    

---

## 8. 監査・署名

- `report.json` に **決定・根拠・ハッシュ** を保存。
    
- 署名済みコミット/マージにより GitHub 上で **Verified** が表示。([GitHub Docs](https://docs.github.com/enterprise-cloud%40latest//articles/signing-commits-with-gpg?utm_source=chatgpt.com "Managing commit signature verification - GitHub Docs"))
    
- SBOM/Provenance 連携や blob 署名には **Cosign の keyless** を推奨。([Sigstore](https://docs.sigstore.dev/cosign/signing/signing_with_blobs/?utm_source=chatgpt.com "Signing Blobs - Sigstore"))
    

---

## 9. エッジケースと運用

- **フォーク PR**：`GITHUB_TOKEN` は既定で **read-only**。必要なら org/repo 設定で write を許可、または Maintainer 手動。([GitHub Docs](https://docs.github.com/en/enterprise-cloud%40latest/actions/reference/github_token-reference?utm_source=chatgpt.com "GITHUB_TOKEN reference - GitHub Enterprise Cloud Docs"))
    
- **Auto-merge の有効化**：必要なら GraphQL の `enablePullRequestAutoMerge` を使用（REST に相当 API なし）。([GitHub](https://github.com/orgs/community/discussions/24719?utm_source=chatgpt.com "Enabling auto merge on a PR using the API - GitHub"))
    
- **rebase/squash の意味**：UI/REST と同等の挙動に準拠。([GitHub Docs](https://docs.github.com/en/enterprise-server%403.17/rest/guides/using-the-rest-api-to-interact-with-your-git-database?utm_source=chatgpt.com "Using the REST API to interact with your Git database - GitHub ..."))
    

---

## 10. サンプル `.runeseal/report.json`（抜粋）

```jsonc
{
  "verdict": "approved",
  "reasons": ["required_checks:pass", "quality:coverage=0.83>=0.75"],
  "signature": "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...",
  "hash": "sha256:5f2e...b1",
  "plan": {
    "checks": [{"name":"ci/test","status":"success"}],
    "reviews": [{"user":"bot","state":"APPROVED"}],
    "diff_metrics": {"loc_add": 124, "loc_del": 9, "files": 6}
  }
}
```