# 防御戦略

インシデント分析から導出した多面的な防御戦略を整理する。

---

## 1. 個人トークン・認証情報の保護

### 1.1 GitHub Fine-grained PAT への移行 (最優先)

Classic PATは全リポジトリ・全組織にアクセス可能なため、1つの漏洩が壊滅的な影響を持つ。

**必須アクション:**
- Classic PATを全て棚卸し → Fine-grained PATに置き換え
- リポジトリ単位でアクセスを制限
- 最小権限の原則（50以上の粒度の高いパーミッションから必要最小限を選択）
- 有効期限の設定（組織オーナーが最大有効期限を強制可能）
- 承認フロー: 組織リソースへのアクセスにはオーナーの承認が必要

### 1.2 npm トークンセキュリティ

- **Trusted Publishing (OIDC)** を最優先で導入: GitHub Actions/GitLab CI/CDからトークンレスで公開
- 書き込みトークン: デフォルト7日、最大90日の有効期限
- Classic tokenは2025年12月に完全廃止済み
- `npm login` は2時間で失効するセッショントークンに変更済み

### 1.3 2FA の強化

- TOTP (時間ベースワンタイムパスワード) はフィッシングに脆弱 → chalk/debug事件で実証
- **WebAuthn/FIDO2 (YubiKey等のハードウェアキー)** がフィッシング耐性のある唯一の選択肢
- 管理権限を持つアカウントには必ずハードウェアキーを設定

### 1.4 Git Credential Helper

- `credential.helper = store` (平文保存) は**絶対に使わない**
- Linux: `libsecret` または `credential-manager` を推奨
- macOS: `osxkeychain`
- パスワードマネージャー連携 (1Password CLI等) も有効

---

## 2. シークレット検出と漏洩防止

### 2.1 Pre-commit フック (Gitleaks)

コミット前にシークレットをローカルで検出する。

```bash
# インストール
brew install gitleaks  # or go install github.com/gitleaks/gitleaks/v8@latest

# pre-commit hookとして設定
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.x.x
    hooks:
      - id: gitleaks
```

### 2.2 CI/CD パイプラインでのスキャン (TruffleHog)

Gitleaksより深く、コード以外(S3, Docker等)もスキャン。検出したシークレットが有効かどうかの**検証機能**あり。

### 2.3 GitHub Push Protection

- プッシュ時にシークレットを検出し、**プッシュ自体をブロック**
- 組織/リポジトリレベルで有効化
- Delegated bypass: バイパスには指定レビュアーの承認が必要

---

## 3. GitHub Actions の保護

### 3.1 SHAピン留め (最重要)

タグは書き換え可能。SHAでのピン留め以外に安全な方法はない。

```yaml
# ❌ 危険: タグは書き換え可能
- uses: actions/checkout@v4

# ✅ 安全: SHAは書き換え不可能
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
```

Renovatebot や Dependabot でSHAピン留めの自動更新が可能。

### 3.2 `pull_request_target` の制限

- 外部PRからのコード実行を許可しない
- やむを得ず使う場合は、ラベルベースのゲーティングを実施
- PRの内容（ブランチ名、コミットメッセージ等）をワークフローに渡さない

### 3.3 GITHUB_TOKEN のスコープ最小化

```yaml
permissions:
  contents: read  # 必要最小限
```

### 3.4 ネットワーク監視

- StepSecurity Harden-Runner で外部通信を監視
- tj-actions事件はこれで発見された

---

## 4. 依存関係管理

### 4.1 ロックファイルの厳格な管理

- `package-lock.json` / `yarn.lock` をコミットし、CI/CDで `npm ci` を使用
- ロックファイルの差分を**必ずレビュー**する

### 4.2 ライフサイクルスクリプトの制御

```bash
# グローバルでpostinstall等を無効化
npm config set ignore-scripts true

# 必要なパッケージのみ許可 (.npmrc)
# npm v9+
# package.json の "scripts" セクションを明示的にレビュー
```

### 4.3 依存関係の監査

```bash
# 脆弱性チェック
npm audit

# サプライチェーンリスクの可視化
npx socket # Socket.dev CLI
```

### 4.4 新しい依存関係追加時のチェックリスト

- [ ] パッケージの人気度・メンテナ数を確認
- [ ] 最近のバージョン履歴に不審な変更がないか
- [ ] `postinstall` / `preinstall` スクリプトの有無と内容を確認
- [ ] 単一メンテナのパッケージは代替を検討
- [ ] スコープ付きパッケージ (@org/package) を優先

---

## 5. 複数組織管理のリスク軽減

### 5.1 アクセス権の分離

- 組織ごとに異なるPATを使用（Fine-grained PATでリポジトリ単位に制限）
- 管理権限を持つアカウントのセキュリティを最優先で強化
- 組織オーナー数を厳格に制限

### 5.2 Enterprise機能の活用

- Enterprise Security Manager (ESM) ロール: 全組織横断でセキュリティアラート・設定を一元管理
- Enterprise Teams: 一度定義すれば複数組織に適用可能

### 5.3 監査と可視化

- 全組織のPAT・トークンを定期的に棚卸し
- 不要なトークンの即座の無効化
- GitGuardian等でシークレットの漏洩を継続監視

---

## 6. IDE拡張機能の保護

- 拡張機能のインストール前にパブリッシャーを確認
- Open VSX使用時は特に注意（名前空間の不整合リスク）
- 定期的に不要な拡張機能を削除
- 拡張機能の自動更新を慎重に管理

---

## 7. AI開発ツールの保護 (新興リスク)

- MCPサーバーの導入前にセキュリティ監査を実施
- AIエージェントに付与するPATのスコープを最小限に
- パブリックリポのIssue/PR内容をAIエージェントが自動処理する際のサンドボックス化
- AIコーディングアシスタントの出力を**必ず人間がレビュー**

---

## 防御の優先度マトリクス

| 優先度 | 対策 | 対象攻撃 | 導入コスト |
|--------|------|----------|-----------|
| **最高** | Fine-grained PAT移行 | アカウント乗っ取り, トークン漏洩 | 低 |
| **最高** | GitHub Actionsのピン留め | タグポイズニング | 低 |
| **最高** | ハードウェアセキュリティキー | フィッシング | 中 |
| **高** | Pre-commitシークレット検出 | トークン漏洩 | 低 |
| **高** | Push Protection有効化 | トークン漏洩 | 低 |
| **高** | npm Trusted Publishing | パッケージ乗っ取り | 中 |
| **中** | ライフサイクルスクリプト制御 | 悪意あるパッケージ | 低 |
| **中** | CI/CDパイプラインスキャン | シークレット漏洩 | 中 |
| **中** | IDE拡張機能の監査 | 拡張機能攻撃 | 低 |
| **低** | AIツールのサンドボックス化 | AIツールチェーン攻撃 | 高 |
