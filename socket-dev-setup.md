# Socket.dev 導入・設定ガイド

Organization全体でSocket.devを有効化し、PRマージ時の必須チェックとして設定する手順。

---

## 前提

- Socket.dev GitHub App が under-the-bridge-hq org の全リポジトリで有効化済み (2026-03-31)
- Socket.dev はPRに対して自動でステータスチェックを付与する

---

## 1. 個別リポジトリでの必須チェック設定

### 方法A: Branch Protection Rules（リポジトリ単位）

1. リポジトリの **Settings** → **Branches** を開く
2. **Branch protection rules** → **Add rule** (既存ルールがあれば Edit)
3. **Branch name pattern**: `main`
4. 以下を有効化:
   - [x] **Require a pull request before merging**
   - [x] **Require status checks to pass before merging**
     - **Status checks that are required**: 検索窓に `socket` と入力
     - **Socket Security** を選択して追加
   - [x] **Require branches to be up to date before merging** (推奨)
5. **Save changes**

### 方法B: Organization Rulesets（org全体一括）

org全リポジトリに一括適用する場合:

1. **Organization Settings** → **Code and automation** → **Repository** → **Rulesets**
2. **New ruleset** → **New branch ruleset**
3. 設定:
   - **Ruleset name**: `socket-security-required`
   - **Enforcement status**: Active
   - **Target repositories**: All repositories（または特定リポジトリを選択）
   - **Target branches**: Default branch
4. **Branch rules** で以下を追加:
   - [x] **Require status checks to pass**
     - **Add checks** → `Socket Security` を追加
5. **Create**

---

## 2. Socket.dev の動作確認

### PRでの確認

Socket.dev が正しく動作していると、PRに以下が表示される:

- **Checks タブ**: `Socket Security` のステータス (✓ passed / ✗ failed)
- **PR コメント**: 新しい依存関係が追加された場合、リスク分析結果がコメントとして投稿される

### 検出されるリスクの例

PRで `package.json` や `package-lock.json` を変更すると、Socket.dev は以下を検出:

| リスク | 説明 | ブロック |
|--------|------|---------|
| Known malware | 既知のマルウェアパッケージ | Yes |
| Install scripts | preinstall/postinstallスクリプトの追加 | Warning |
| Typosquatting | 正規パッケージに類似した名前 | Yes |
| Obfuscated code | 難読化されたコード | Warning |
| Network access | 予期しないネットワーク通信 | Warning |
| Shell access | シェルコマンド実行 | Warning |
| Telemetry | テレメトリ/トラッキングコード | Info |

### チェックが表示されない場合

- PRで `package.json` / `package-lock.json` / `yarn.lock` の変更がない場合、チェックはスキップされる
- GitHub App がリポジトリにインストールされているか確認: `Organization Settings` → `GitHub Apps` → `Socket Security`

---

## 3. Socket.dev ポリシー設定（オプション）

Socket.dev の Web ダッシュボード (https://socket.dev) でポリシーをカスタマイズできる:

1. https://socket.dev にGitHubアカウントでログイン
2. **Organization** → **under-the-bridge-hq** を選択
3. **Settings** → **Policies** で以下を設定:
   - どのリスクレベルでPRをブロックするか
   - 特定パッケージの例外設定
   - 通知設定

---

## 4. CLI での利用（ローカル開発時）

```bash
# インストール
npm install -g @socketsecurity/cli

# パッケージ追加前に安全性を確認
socket npm install <package-name>

# プロジェクト全体のスキャン
socket scan create ./

# CI向け（ポリシー違反があればexit 1）
socket ci ./
```

API トークンは https://socket.dev/dashboard → **Settings** → **API Tokens** で生成。

---

## 5. 対象リポジトリチェックリスト

| リポジトリ | npm/依存関係あり | Socket必須化 | 設定済み |
|-----------|-----------------|-------------|---------|
| ai-steward | Yes | **必須** | [ ] |
| defend-against-supply-chain-attacks | No (ドキュメントのみ) | 不要 | - |
| （他のリポジトリを追加） | | | [ ] |

### ai-steward での設定手順

```
1. https://github.com/under-the-bridge-hq/ai-steward/settings/branches
2. main ブランチの Branch protection rule を編集
3. Require status checks → Socket Security を追加
4. Save
```

---

## 6. 他のセキュリティチェックとの組み合わせ（推奨）

Socket.dev に加えて、以下もrequired checkとして追加を推奨:

```yaml
# .github/workflows/package-security.yml
name: Package Security
on:
  pull_request:
    paths:
      - 'package.json'
      - 'package-lock.json'

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1

      - uses: actions/setup-node@60edb5dd545a775178f52524783378180af0d1f8 # v4.0.2
        with:
          node-version: '20'

      # lockfile整合性検証
      - name: Lint lockfile
        run: |
          npx lockfile-lint \
            --path package-lock.json \
            --type npm \
            --allowed-hosts npm \
            --validate-https \
            --validate-integrity

      # レジストリ署名検証
      - name: Verify registry signatures
        run: npm audit signatures

      # 既知脆弱性チェック
      - name: Audit dependencies
        run: npm audit --audit-level=high
        continue-on-error: true  # 初期導入時はwarningのみ
```

これもrequired checkに追加すると、Socket.dev (行動分析) + lockfile-lint (整合性) + npm audit (既知CVE) + npm audit signatures (署名) の多層防御になる。
