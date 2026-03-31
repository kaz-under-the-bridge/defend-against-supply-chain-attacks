# 攻撃ベクトル分類

インシデントから抽出した攻撃手法を体系的に分類する。

---

## 1. パッケージレジストリ攻撃

### 1.1 メンテナアカウント乗っ取り

最も破壊力の高い攻撃ベクトル。単一メンテナの侵害が数十億DLに影響する。

| 手法 | 実例 |
|------|------|
| フィッシング (偽2FAリセット) | chalk/debug (2025年9月) - `npmjs.help` 偽ドメイン |
| アカウント乗っ取り (メール変更) | Axios (2026年3月) - ProtonMailに変更 |
| トークン窃取 → 自動拡散 | Shai-Hulud (2025年9-11月) - ワーム型 |

**ポイント**: TOTP方式の2FAはフィッシングに脆弱。WebAuthn/FIDO2が必要。

### 1.2 タイポスクワッティング

正規パッケージに似た名前で偽パッケージを公開する。

- 2025年7月: TypeScript, discord.js, ethers.js等を模倣した10パッケージ (合計9,900DL)
- 2026年2月: SANDWORM_MODE - 19のタイポスクワッティングパッケージ
- TensorFlow.jsを模倣したAI/ML開発者標的パッケージ

### 1.3 悪意あるメンテナ (長期潜伏)

正規パッケージとして運用を開始し、信頼獲得後にマルウェア化する。

- @0xengine/xmlrpc: 1年以上の正規運用後にマルウェア化
- XZ Utils: 2年間のソーシャルエンジニアリングでメンテナ権限を獲得

### 1.4 レジストリスパム

- tea.xyzプロトコルを悪用した15万パッケージのトークンファーミング

---

## 2. CI/CD パイプライン攻撃

### 2.1 GitHub Actionsタグポイズニング

ミュータブルなバージョンタグを悪意あるコミットに書き換える。

| 事件 | 影響 |
|------|------|
| tj-actions/changed-files | 23,000リポジトリ |
| reviewdog/action-setup | tj-actionsへのカスケード侵害の起点 |
| Trivy Action | 10,000以上のワークフロー |

**根本原因**: GitHub Actionsのタグはミュータブル。SHAピン留め以外に安全な方法はない。

### 2.2 `pull_request_target` 悪用 (Pwn Request)

外部PRからシークレットにアクセス可能なワークフローを悪用する。

- Ultralytics (2024年12月): ブランチ名操作でPyPIトークン窃取
- nx (2025年8月): AWS管理者権限の奪取
- Trivy (2026年2-3月): PATの窃取

### 2.3 CI/CDバイパス

正規のビルドパイプラインを迂回してレジストリに直接公開する。

- Axios: GitHub Actions CI/CDを完全にバイパスし、npmに直接公開
- XZ Utils: リリースアーティファクトにのみバックドアを注入（ソースコードには含まない）

---

## 3. IDE拡張機能攻撃

### 3.1 マーケットプレイス名前空間の不整合

- Cursor/Windsurf等がOpen VSXを使用 → VS Code Marketplaceとの名前空間不一致を悪用

### 3.2 依存関係チェーン悪用

- GlassWorm: 信頼獲得後、更新で悪意ある依存拡張機能を追加
- 19 VS Code拡張機能: 依存関係フォルダ内にマルウェアを隠蔽

### 3.3 偽拡張機能

- prettier-vscode-plus: 偽Prettier → マルチステージRAT
- Ethcode: 大規模PRに2行の悪意あるコードを紛れ込ませる

---

## 4. AIツールチェーン攻撃 (新興)

### 4.1 MCPサーバー注入

- SANDWORM_MODE: MCPサーバーを注入し、AIコーディングアシスタントにプロンプトインジェクション
- Postmark MCP: バックドアにより全送信メールが攻撃者にBCC

### 4.2 AIエージェントハイジャック

- GitHub MCP統合: パブリックリポの悪意あるIssueでAIエージェントを操作し、プライベートリポからデータ窃取

---

## 5. ライフサイクルスクリプト悪用

ほぼ全ての攻撃で `preinstall` / `postinstall` スクリプトが悪用されている。

| スクリプト | 悪用例 |
|-----------|--------|
| `preinstall` | Shai-Hulud (自己増殖の起点) |
| `postinstall` | Axios (RAT配布), nx (クレデンシャル窃取), タイポスクワッティング各種 |

---

## 6. カスケード（連鎖）攻撃パターン

一つの侵害が連鎖的に拡大するパターンが増加している。

```
reviewdog/action-setup
  → tj-actions/changed-files
    → Coinbase agentkit (最終標的)

spotbugs (pull_request_target)
  → reviewdog メンテナPAT窃取
    → tj-actions-bot PAT窃取

Shai-Hulud
  → npmトークン窃取
    → 被害者の他パッケージに自動感染 (最大100パッケージ)
      → さらに次の被害者のトークンを窃取...

TeamPCP キャンペーン (2026年2-3月) ← 最も深刻なカスケード事例
  Phase 1: Trivy GitHub Actionの pull_request_target 悪用でPAT窃取
    → Phase 2: Trivyタグポイズニング (v0.69.4-6) → Docker Hub/GHCR/ECRに配布
      → Phase 3: Trivyを使うCI/CDパイプラインからPyPIトークン窃取
        → Phase 9: LiteLLM PyPI侵害 (46分間で47,000 DL)
          → .pthファイルでPython起動時に自動実行
            → Kubernetes横展開 (CanisterWorm)
```

---

## 攻撃ベクトル別の発生頻度と影響度マトリクス

| 攻撃ベクトル | 発生頻度 | 影響度 | 検出難易度 |
|-------------|---------|--------|-----------|
| メンテナアカウント乗っ取り | 高 | 極高 | 中 |
| タイポスクワッティング | 極高 | 低〜中 | 低 |
| GitHub Actionsタグポイズニング | 中 | 極高 | 中 |
| pull_request_target悪用 | 中 | 高 | 高 |
| 長期潜伏型 | 低 | 極高 | 極高 |
| IDE拡張機能 | 高 | 中〜高 | 中 |
| AIツールチェーン | 増加中 | 不明 | 高 |
| ライフサイクルスクリプト悪用 | 極高 | 高 | 低 |
