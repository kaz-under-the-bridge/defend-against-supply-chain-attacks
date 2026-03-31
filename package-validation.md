# パッケージ正当性検証と防御実装

vibe codingやAIコーディング時代において、悪意あるパッケージの混入を防ぐための実践的な検証手法と実装ガイド。

---

## 混入経路

| 経路 | 説明 | 例 |
|------|------|---|
| **Slopsquatting** | AIが存在しないパッケージ名を「幻覚」で提案 → 攻撃者が先回りして同名パッケージを登録 | AIが `husky-config-utils` を提案、実在せず攻撃者が登録 |
| **タイポスクワッティング** | AIが正規パッケージ名をtypoして提案 | `lodash` → `lodassh` |
| **MCPプロンプトインジェクション** | AIがMCPサーバー経由で操作され、悪意あるパッケージを推薦 | SANDWORM_MODE (2026年2月) |
| **悪意あるPR** | 大量変更に紛れて悪意ある依存関係を追加 | Ethcode VS Code (2025年6月) |

---

## 1. パッケージ正当性チェックツール

### 1.1 Socket.dev（最推奨）

`npm audit` が既知CVEしか検出しないのに対し、Socket.devは**行動分析**で未知の脅威を検出する。

```bash
# インストール
npm install -g @socketsecurity/cli

# 安全なnpm install（インストール前にチェック）
socket npm install <package-name>

# プロジェクト全体のスキャン
socket scan create ./my-project

# CI向け（ポリシー違反があればexit 1）
socket ci ./my-project
```

**Webで個別パッケージを確認:**
`https://socket.dev/npm/package/<package-name>`

**検出項目（70以上）:**

| カテゴリ | 検出内容 |
|---------|---------|
| サプライチェーンリスク | タイポスクワッティング、依存性混乱、マルウェア |
| コード行動分析 | インストールスクリプト、難読化コード、eval()使用 |
| 特権API使用 | シェルコマンド実行、ファイルシステムアクセス、環境変数読み取り |
| メタデータ分析 | テレメトリ/トラッキング、ネイティブコード |
| 品質/メンテナンス | 放棄パッケージ、ライセンス問題、既知脆弱性 |

**Socket vs npm audit vs Snyk:**

| 機能 | npm audit | Snyk | Socket.dev |
|------|----------|------|-----------|
| 既知CVE検出 | Yes | Yes | Yes |
| ゼロデイ検出 | No | 限定的 | **Yes（行動分析）** |
| マルウェア検出 | No | No | **Yes** |
| タイポスクワッティング | No | No | **Yes** |
| 検出タイミング | インストール後 | インストール後 | **インストール前** |

**料金**: OSSプロジェクトは無料。個人プライベートリポ1つ無料。

**GitHub App**: https://github.com/apps/socket-security をインストールするだけで、PRごとに依存関係の変更を自動分析。

### 1.2 npm audit signatures（レジストリ署名検証）

```bash
# npmレジストリが付与するECDSA署名を検証
npm audit signatures
```

署名がないパッケージ = npmレジストリ以外から公開された可能性がある。Sigstoreによる来歴（provenance）証明書も検証。

### 1.3 OpenSSF Scorecard（パッケージ健全性スコア）

```bash
# インストール
brew install scorecard  # or go install github.com/ossf/scorecard/v5/cmd/scorecard@latest

# npmパッケージを直接チェック
scorecard --npm=express --show-details

# GitHubリポジトリで確認
scorecard --repo=github.com/expressjs/express

# JSON出力
scorecard --repo=github.com/expressjs/express --format=json
```

0-10のスコアで評価: メンテナンス活発度、コードレビュー有無、ブランチ保護、署名リリース、依存関係ピン留め等。

### 1.4 lockfile-lint（ロックファイル整合性検証）

```bash
npx lockfile-lint \
  --path package-lock.json \
  --type npm \
  --allowed-hosts npm \
  --validate-https \
  --validate-integrity
```

検出: レジストリ変更、HTTP平文取得、integrityハッシュ不整合、不正URLスキーム。

---

## 2. postinstallスクリプト無効化（即時導入推奨）

ほぼ全てのnpmサプライチェーン攻撃で `preinstall` / `postinstall` スクリプトが悪用されている。

### 2.1 基本設定

```ini
# プロジェクトルートの .npmrc
ignore-scripts=true
```

### 2.2 postinstallが必要なパッケージ

| パッケージ | スクリプトの目的 | 無効化時の影響 |
|-----------|-----------------|---------------|
| esbuild | プラットフォーム別バイナリ配置 | ビルド不能 |
| sharp | libvipsネイティブバイナリDL | 画像処理不能 |
| puppeteer | ChromiumバイナリDL | ブラウザ起動不能 |
| bcrypt | ネイティブC++コンパイル | ハッシュ関数不能 |
| sqlite3 | ネイティブバインディングビルド | DB接続不能 |
| node-sass | libsassコンパイル | Sassコンパイル不能 |
| canvas | Cairo/Pangoバインディング | Canvas API不能 |

### 2.3 選択的リビルドワークフロー

```bash
# Step 1: スクリプト無効でインストール
npm ci --ignore-scripts

# Step 2: 必要なパッケージだけ個別にリビルド
npm rebuild esbuild sharp bcrypt

# Step 3: Puppeteerはブラウザ別途DL
npx puppeteer browsers install chrome
```

`package.json` に定義:

```json
{
  "scripts": {
    "ci:install": "npm ci --ignore-scripts && npm rebuild esbuild sharp bcrypt"
  }
}
```

### 2.4 @lavamoat/allow-scripts（allowlist管理）

npmで唯一の実用的なallowlistソリューション:

```bash
npm i -D @lavamoat/allow-scripts

# 初期化 + 現状検出
npx allow-scripts setup
npx allow-scripts auto
```

`package.json` に生成される設定:

```json
{
  "lavamoat": {
    "allowScripts": {
      "esbuild": true,
      "sharp": true,
      "bcrypt": true,
      "core-js": false,
      "husky": false
    }
  }
}
```

```bash
# 許可されたスクリプトのみ実行
npx allow-scripts run
```

### 2.5 判断フロー

```
パッケージ追加時:
  1. Socket.devで正当性チェック
  2. ignore-scriptsでインストール
  3. postinstallが必要？
     → Yes: Socketで安全確認済み → allowlistに追加 → rebuild
     → No: そのまま使用
  4. 不明/リスクあり → 代替パッケージを検討
```

---

## 3. lockfile差分レビュー

PRレビュー時に以下を重点チェック:

| フラグ | 意味 | 危険度 |
|--------|------|--------|
| 新規パッケージ追加（特に推移的依存） | 意図しない依存の混入 | 高 |
| レジストリURL変更 | `registry.npmjs.org` 以外からの取得 | 極高 |
| バージョン変更なしのintegrityハッシュ変更 | パッケージ改竄の兆候 | 極高 |
| メジャーバージョンジャンプ | 予期しない大幅変更 | 中 |
| 依存ツリーの大幅変動 | 小さなバージョン変更で大量の推移的依存が変わる | 中 |

### CIでの自動チェック

```yaml
# lockfile変更のPRコメント表示
- uses: codepunkt/npm-lockfile-changes@v1
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    failOnDowngrade: true
```

---

## 4. SBOM生成と継続監視

### CycloneDX形式でのSBOM生成（推奨）

```bash
# @cyclonedx/cyclonedx-npmを使用（npm sbomより正確）
npx @cyclonedx/cyclonedx-npm --omit dev --validate --output-file sbom.json
```

### CIパイプラインでの保管

```yaml
- name: Generate SBOM
  run: npx @cyclonedx/cyclonedx-npm --omit dev --validate --output-file sbom.json
- uses: actions/upload-artifact@v4
  with:
    name: sbom
    path: sbom.json
```

---

## 5. CI パイプライン統合（総合例）

```yaml
name: Package Security Check
on:
  pull_request:
    paths:
      - 'package.json'
      - 'package-lock.json'

permissions:
  pull-requests: write
  contents: read

jobs:
  package-security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1

      - uses: actions/setup-node@60edb5dd545a775178f52524783378180af0d1f8 # v4.0.2
        with:
          node-version: '20'

      # 1. lockfile整合性検証
      - name: Lint lockfile
        run: |
          npx lockfile-lint \
            --path package-lock.json \
            --type npm \
            --allowed-hosts npm \
            --validate-https \
            --validate-integrity

      # 2. スクリプト無効で整合性チェック
      - name: Verify lockfile integrity
        run: npm ci --ignore-scripts

      # 3. 既知脆弱性チェック
      - name: Audit dependencies
        run: npm audit --audit-level=high

      # 4. レジストリ署名検証
      - name: Verify registry signatures
        run: npm audit signatures

      # 5. Socket.dev行動分析（要SOCKET_TOKEN）
      - name: Socket security check
        if: env.SOCKET_TOKEN != ''
        env:
          SOCKET_TOKEN: ${{ secrets.SOCKET_TOKEN }}
        run: npx socket ci ./

      # 6. SBOM生成
      - name: Generate SBOM
        run: npx @cyclonedx/cyclonedx-npm --omit dev --validate --output-file sbom.json

      - uses: actions/upload-artifact@65462800fd760344b1a7b4382951275a0abb4808 # v4.3.3
        with:
          name: sbom
          path: sbom.json
```

**注**: GitHub Actionsはすべて**SHAピン留め**で指定。

---

## 実践チェックリスト（コピペ用コマンド集）

```bash
# ===== 開発時：新しいパッケージを追加する前 =====

# 1. Socket.devで正当性チェック
socket npm install <package-name>
# または Web: https://socket.dev/npm/package/<package-name>

# 2. OpenSSF Scorecardで健全性確認
scorecard --npm=<package-name> --show-details

# ===== プロジェクト全体の定期チェック =====

# 3. レジストリ署名検証
npm audit signatures

# 4. 既知脆弱性チェック
npm audit

# 5. lockfile整合性
npx lockfile-lint --path package-lock.json --allowed-hosts npm --validate-https --validate-integrity

# 6. Node.jsランタイム自体の脆弱性
npx is-my-node-vulnerable

# 7. SBOM生成
npx @cyclonedx/cyclonedx-npm --omit dev --validate --output-file sbom.json
```

---

## 参考

- [Socket.dev CLI](https://github.com/SocketDev/socket-cli) / [Docs](https://docs.socket.dev/)
- [Socket.dev GitHub App](https://github.com/apps/socket-security)
- [OpenSSF Scorecard](https://scorecard.dev/)
- [lockfile-lint](https://github.com/lirantal/lockfile-lint)
- [@lavamoat/allow-scripts](https://lavamoat.github.io/guides/allow-scripts/)
- [@cyclonedx/cyclonedx-npm](https://github.com/CycloneDX/cyclonedx-node-npm)
- [npm audit signatures](https://docs.npmjs.com/verifying-registry-signatures/)
- [is-my-node-vulnerable](https://github.com/nodejs/is-my-node-vulnerable)
- [ossf/malicious-packages DB](https://github.com/ossf/malicious-packages)
- [npm lockfile changes Action](https://github.com/marketplace/actions/npm-lockfile-changes)
