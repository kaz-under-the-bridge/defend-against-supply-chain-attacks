# GHEC セキュリティガードレール設計

## 1. 背景と目的

### IaC管理によるガードレールの意義

GitHub Enterprise Cloud (GHEC) のOrganization/Enterprise設定は、開発チーム全体のセキュリティベースラインを決定する。これらの設定を手動管理すると、設定ドリフト（意図しない変更の蓄積）が発生し、セキュリティポリシーが形骸化するリスクがある。

Terraformによる宣言的管理を行うことで:
- 設定の変更が必ずPR + レビューを経由する
- 変更履歴がgitに残り、監査可能になる
- ドリフト検出により手動変更を即座に発見できる
- 新リポジトリ作成時にセキュリティ設定が自動適用される

### AI開発時代のハーネスエンジニアリング

AIコーディングアシスタント（Claude Code, GitHub Copilot等）が日常的にコードを生成・修正する時代において、AIの行動に対するガードレール（ハーネス）の設計が重要になる。

AIは以下のリスクを内包する:
- **幻覚パッケージの提案（Slopsquatting）**: 存在しないパッケージ名を提案し、攻撃者がその名前を先回りで登録
- **マイナーなActionsの無断追加**: 検証されていないサードパーティActionsをワークフローに追加
- **設定の過剰な緩和**: 「動かすため」にセキュリティ設定を無効化

GHECのガードレールは、AIの出力が本番環境に到達する前に人間の審査を強制するチェックポイントとして機能する。例えば、Actions許可リストに登録されていないActionをAIが追加した場合、CIが失敗し、人間がそのActionの安全性を審査するフローが自然に生まれる。

---

## 2. 脅威モデル

### 2.1 サプライチェーン攻撃の主要パターン

| 分類 | 攻撃手法 | 代表的インシデント |
|------|----------|-------------------|
| **パッケージレジストリ攻撃** | メンテナアカウント乗っ取り | Axios (2026/3), chalk/debug (2025/9) |
| | タイポスクワッティング | SANDWORM_MODE (2026/2) |
| | 長期潜伏型（正規→マルウェア化） | XZ Utils (2024/3) |
| **CI/CDパイプライン攻撃** | GitHub Actionsタグポイズニング | tj-actions (2025/3), Trivy Action (2026/2-3) |
| | `pull_request_target` 悪用 (Pwn Request) | Ultralytics (2024/12) |
| **IDE拡張機能攻撃** | マーケットプレイス名前空間の不整合 | Cursor/Windsurf + Open VSX |
| | 依存関係チェーン悪用 | GlassWorm (72拡張機能) |
| **AIツールチェーン攻撃** | MCPサーバー注入（プロンプトインジェクション） | SANDWORM_MODE, Postmark MCP |
| | Slopsquatting（AIの幻覚パッケージ） | 新興リスク |
| **ライフサイクルスクリプト悪用** | preinstall/postinstallによるコード実行 | ほぼ全てのnpm攻撃 |
| **カスケード攻撃** | 1つの侵害から連鎖的に拡大 | TeamPCP: Trivy→LiteLLM→K8s |

### 2.2 AI開発固有のリスク

| リスク | 説明 | 影響 |
|--------|------|------|
| Slopsquatting | AIが存在しないパッケージ名を提案→攻撃者が先回り登録 | 悪意あるパッケージのインストール |
| Actions無断追加 | AIが未検証のサードパーティActionsをワークフローに追加 | CI/CDパイプラインの侵害 |
| 設定緩和 | AIが「エラー解消」のためセキュリティ設定を無効化 | ガードレールの形骸化 |
| シークレット混入 | AIがコード内にトークンやキーを直接記述 | 認証情報の漏洩 |

### 2.3 内部脅威

| リスク | 説明 | 対策 |
|--------|------|------|
| トークン漏洩 | Classic PATが1つ漏洩すると全組織が危険 | Fine-grained PAT強制 |
| 設定ドリフト | UI経由の手動変更がTerraformと乖離 | ドリフト検出 + IaC強制 |
| 権限の過剰付与 | 不要なadmin権限の蓄積 | 最小権限の原則 + 定期レビュー |

---

## 3. GHEC設定によるガードレール一覧

### 3.1 Enterprise レベル（手動設定）

| 設定 | 目的 | 現状 | 目標 |
|------|------|------|------|
| SAML SSO | 認証の一元管理 | **設定済み** | 維持 |
| 2FA強制 | 多要素認証 | **設定済み** | 維持 |
| IdP側MFA（FIDO2推奨） | フィッシング耐性のある認証 | 要確認 | IdPでFIDO2/WebAuthn必須化 |
| Classic PAT無効化 | トークン漏洩の爆発半径縮小 | 未設定 | **Fine-grained PATのみ許可** |
| Fine-grained PAT最大有効期限 | 長期トークンのリスク低減 | 未設定 | **30日** |
| Audit Log Streaming | セキュリティ監視 | 未設定 | 将来対応 |

### 3.2 Organization レベル（Terraform管理）

#### Organization Settings

| 設定 | 目的 | 現状 | 目標 |
|------|------|------|------|
| `members_can_create_public_repositories` | public repo作成禁止 | `true` | **`false`** |
| `members_can_create_repositories` | リポジトリ作成の制限 | `true` | 検討（admin限定にするか） |
| `members_can_fork_private_repositories` | fork禁止 | **`false`** | 維持 |
| `default_repository_permission` | デフォルト権限 | **`read`** | 維持 |

#### GitHub Actions許可リスト

| 設定 | 目的 | 現状 | 目標 |
|------|------|------|------|
| `enabled_repositories` | Actionsを使えるリポジトリ | `all` | `all` |
| `allowed_actions` | 許可するActions | `all`（制限なし） | **`selected`（許可リスト）** |
| `github_owned_allowed` | GitHub公式Actions | - | **`true`** |
| `verified_allowed` | Verified creatorsのActions | - | **`true`** |
| `patterns_allowed` | 明示的に許可するActions | - | **別途定義** |

##### 許可リスト対象（under-the-bridge-hq）

現在使用中のサードパーティActions:
- `tibdex/github-app-token@*`
- `hashicorp/setup-terraform@*`
- `aws-actions/configure-aws-credentials@*`

#### Org Ruleset（全リポジトリ共通）

| ルール | 目的 | 設定値 |
|--------|------|--------|
| `deletion` | デフォルトブランチの削除禁止 | `true` |
| `non_fast_forward` | force push禁止 | `true` |
| `required_linear_history` | リニアな履歴の強制 | `true` |
| `pull_request` | PR必須 | レビュー1名以上 |
| `allowed_merge_methods` | マージ方法の制限 | `["squash"]` |

### 3.3 Repository レベル（Terraform管理）

| 設定 | 目的 | Orgで設定 | Repoで設定 |
|------|------|:---------:|:----------:|
| ブランチ削除禁止 | | **○** | |
| force push禁止 | | **○** | |
| PR必須（最低レビュー数） | | **○** (1名) | ○ (上乗せ可) |
| squash mergeのみ | | **○** | |
| required status checks | CI通過必須 | | **○** |
| required signatures | コミット署名 | 検討 | |

---

## 4. Org Ruleset vs Repository Ruleset — 設計方針

### 原則

```
Org Ruleset:  全リポジトリに適用する最低限のガードレール（例外なし）
Repo Ruleset: リポジトリ固有の要件（status check等）
```

### Org Rulesetで強制するもの

- デフォルトブランチの保護（削除禁止、force push禁止）
- PR必須（最低1名のレビュー）
- squash mergeのみ
- リニアな履歴

これらは全リポジトリで例外なく適用すべきセキュリティベースライン。

### Repo Rulesetで個別設定するもの

- **required status checks**: リポジトリごとにCIジョブ名が異なるため、個別に設定が必要。Orgレベルで設定するとワークフローが存在しないリポジトリでマージ不能になる。
- **追加のレビュー要件**: 重要リポジトリでは2名以上のレビューを要求するなど。

### bypass_actorsの設計

```
Org Ruleset:
  bypass_actors:
    - actor_id: 5 (Repository Admin)
      bypass_mode: "always"    # 緊急時のbypass用

Repo Ruleset:
  bypass_actors:
    - actor_id: 5 (Repository Admin)
      bypass_mode: "always"
    - actor_id: <GitHub App ID>  # CI/CD用
      bypass_mode: "always"
```

**注意**: bypass_actorsを最小限に保つこと。bypass使用はAudit Logに記録される。

---

## 5. GitHub Actions セキュリティポリシー

### 許可リスト管理方針

1. **デフォルト許可**: GitHub公式Actions (`actions/*`, `github/*`) + Verified creators
2. **明示的許可**: Verified creator以外のActionsは `patterns_allowed` に追加が必要
3. **追加フロー**: AIまたは開発者が新しいActionを使いたい場合:
   - CIが失敗（許可リストにないため）
   - PRで人間がActionの安全性を審査
   - 承認後、Terraform PRでActionsを許可リストに追加
   - マージ・適用後に元のPRを再実行

### SHAピン留めの推奨

タグ（`@v2`等）はミュータブルで書き換え可能。tj-actions事件（2025/3）では、タグが悪意あるコミットに書き換えられた。

```yaml
# ❌ タグ参照（書き換え可能）
- uses: tibdex/github-app-token@v2

# ✅ SHAピン留め（イミュータブル）
- uses: tibdex/github-app-token@3beb63f4bd073e61482598c75036e6a2f2c9b5dc # v2.1.0
```

Renovateで自動更新 + SHAピン留めの組み合わせが推奨。

### 許可済みActionsの一覧

| Action | 用途 | 使用リポジトリ |
|--------|------|---------------|
| `tibdex/github-app-token` | GitHub Appトークン生成 | github-admin |
| `hashicorp/setup-terraform` | Terraformセットアップ | github-admin |
| `aws-actions/configure-aws-credentials` | AWS OIDC認証 | github-admin |

---

## 6. 現状と目標状態の差分

### Enterprise レベル

| 設定 | 現状 | 目標 | 対応方法 | 優先度 |
|------|------|------|----------|--------|
| SAML SSO | ✅ 設定済み | 維持 | - | - |
| 2FA強制 | ✅ 設定済み | 維持 | - | - |
| Classic PAT無効化 | ❌ 未設定 | Fine-grained PATのみ | Enterprise管理コンソール | **高** |
| PAT最大有効期限 | ❌ 未設定 | 30日 | Enterprise管理コンソール | **高** |
| IdP MFA種別 | ❓ 要確認 | FIDO2/WebAuthn | IdP設定 | 中 |
| Audit Log Streaming | ❌ 未設定 | 外部転送 | gh API | 低 |

### Organization レベル

| 設定 | 現状 | 目標 | 対応方法 | 優先度 |
|------|------|------|----------|--------|
| Public repo作成禁止 | ❌ `true` | `false` | **Terraform** | **高** |
| Actions許可リスト | ❌ 制限なし | selected | **Terraform** | **高** |
| Org Ruleset | ❌ なし | 全リポジトリ共通ルール | **Terraform** | **高** |
| Fork禁止 | ✅ `false` | 維持 | - | - |
| デフォルト権限 `read` | ✅ 設定済み | 維持 | - | - |

### Repository レベル

| 設定 | 現状 | 目標 | 対応方法 | 優先度 |
|------|------|------|----------|--------|
| Repo Ruleset (status check) | ✅ github-admin済み | 全リポジトリ | **Terraform** | 中 |
| GHAS (Secret Scanning等) | ❌ ライセンス未確認 | 将来対応 | Terraform (GHAS後) | 低 |

---

## 7. 実装優先度

### Phase 1: 即時適用（Terraform PR）

- [ ] `members_can_create_public_repositories = false`
- [ ] GitHub Actions許可リスト（verified + 明示的許可3つ）
- [ ] Org Ruleset（削除禁止、FF禁止、PR必須、squash mergeのみ）

### Phase 2: Enterprise管理コンソール設定

- [ ] Classic PAT無効化 / Fine-grained PAT強制
- [ ] Fine-grained PATの最大有効期限を30日に設定
- [ ] IdP側MFA設定の確認と強化

### Phase 3: GHAS導入後

- [ ] Secret Scanning + Push Protection全リポ有効化
- [ ] Dependabot Alerts + Security Updates全リポ有効化
- [ ] Code Scanning（CodeQL）デフォルトセットアップ

---

## 8. GHECガードレールの限界と補完策

GHECの設定だけでは防げない脅威領域が存在する。これらはGitHubの外側で補完する必要がある。

### 8.1 パッケージレジストリ層の脅威

| 脅威 | GHECで防げない理由 | 補完策 |
|------|-------------------|--------|
| **タイポスクワッティング** | npmレジストリ側の問題 | Socket.dev（行動分析）、パッケージ追加時の手動チェック |
| **Slopsquatting** | AIの幻覚はGitHub外で発生 | lockfile差分レビュー、`npm audit signatures`、Socket.dev |
| **ライフサイクルスクリプト悪用** | パッケージマネージャの仕様 | `ignore-scripts=true` + `@lavamoat/allow-scripts`（許可リスト） |
| **レジストリ直接公開** | CI/CDをバイパスしてnpmに直接公開 | npm Trusted Publishing (OIDC)、npm audit signatures |
| **長期潜伏型攻撃** | 正規パッケージが徐々にマルウェア化 | Socket.devの行動分析、SBOM生成と継続監視 |

### 8.2 開発環境層の脅威

| 脅威 | GHECで防げない理由 | 補完策 |
|------|-------------------|--------|
| **IDE拡張機能攻撃** | GitHub外（VS Code Marketplace, Open VSX） | 拡張機能の定期監査、信頼できるソースのみ使用、拡張機能の権限レビュー |
| **AIツールチェーン攻撃（MCP注入）** | GitHub外（MCPサーバー、AIエージェント） | MCPサーバーの権限最小化、ツール許可リスト、プロンプトインジェクション対策 |
| **AIエージェントハイジャック** | AIエージェントの動作はGitHub管轄外 | エージェントの権限制限、操作ログの監査、人間の承認フロー |

### 8.3 カスケード攻撃への対応

カスケード攻撃（1つの侵害が連鎖的に拡大する攻撃、例: TeamPCPキャンペーン）は、単一の防御策では防げない。

多層防御の考え方:
1. **予防**: Actions許可リスト、SHAピン留め、パッケージ監査
2. **検出**: Socket.dev、Audit Log監視、ドリフト検出
3. **封じ込め**: Fine-grained PATによる爆発半径の縮小、最小権限の原則
4. **回復**: Terraformによる設定の宣言的復元、インシデント対応手順

### 8.4 補完ツールの導入優先度

| ツール | 目的 | 優先度 | 備考 |
|--------|------|--------|------|
| Socket.dev | パッケージの行動分析 | 高 | npm/PyPIの脅威に対する最も有効な対策 |
| Gitleaks (pre-commit) | ローカルでのシークレット検出 | 高 | コミット前にブロック |
| lockfile-lint | lockfile差分の自動検証 | 中 | レジストリURL変更の検出 |
| SBOM生成 | ソフトウェア構成の可視化 | 中 | 長期的な監視基盤 |
| MCPサーバー監査 | AIツールチェーンの安全性確認 | 中 | 新興リスクへの対応 |
