# サプライチェーン攻撃 インシデントカタログ

直近のサプライチェーン攻撃事件を時系列で整理する。特に2024年後半〜2026年3月の主要インシデントを対象とする。

---

## 2026年

### Axios npm パッケージ侵害 (2026年3月30日)

- **CVE**: N/A
- **対象**: `axios@1.14.1`, `axios@0.30.4`
- **攻撃ベクトル**: メンテナアカウント乗っ取り
- **概要**: 週間8,300万DLのHTTPクライアント。メンテナ(jasonsaayman)のnpmアカウントが乗っ取られ、メールがProtonMailに変更された。悪意ある依存関係 `plain-crypto-js@4.2.1` がpackage.jsonに追加され、postinstallでクロスプラットフォームRATを配布。GitHub Actions CI/CDを完全にバイパスし、npmレジストリに直接公開された。
- **影響**: 約3時間で検出・削除。影響環境の3%で実行を確認。
- **発見**: StepSecurityが検知
- **教訓**:
  - CI/CDパイプラインを経由しない直接公開を制限する仕組みが必要
  - ソースコードで一切importされていない依存関係がpostinstallで実行される手口
  - npmアカウントの2FA必須化、長期クラシックトークンのリスク
- **参考**: [StepSecurity](https://www.stepsecurity.io/blog/axios-compromised-on-npm-malicious-versions-drop-remote-access-trojan), [Wiz](https://www.wiz.io/blog/axios-npm-compromised-in-supply-chain-attack)

---

### Trivy サプライチェーン攻撃 - TeamPCP (2026年2月末〜3月19日)

- **CVE**: N/A
- **対象**: `aquasecurity/trivy-action` (75/76バージョンタグ), `aquasecurity/setup-trivy` (全7タグ), Trivy v0.69.4
- **攻撃ベクトル**: GitHub Actions設定ミス → PAT窃取 → タグポイズニング
- **概要**: GitHub Actionsの `pull_request_target` ワークフローの脆弱性でPATを窃取。Aqua Securityが認証情報をローテーションしたが不完全で、攻撃者(TeamPCP)が残存する認証情報で3月19日に再侵害。v0.69.4タグをプッシュし、自動リリースパイプラインがGitHub Releases、Docker Hub、GHCR、Amazon ECRに悪意あるバイナリを配布。偽ドメイン `scan.aquasecurtiy[.]org` へデータ流出。
- **影響**: 10,000以上のダウンストリームワークフロー。AWS/GCP/Azure/Kubernetesクレデンシャル窃取。50以上のファイルシステム箇所からSSH鍵等を収集。
- **発見**: GitHubリリースとタグの不一致
- **教訓**:
  - **セキュリティツール自体がサプライチェーン攻撃の対象になる**
  - 認証情報ローテーションは「完全」でなければ無意味
  - GitHub Actionsのタグはミュータブル → SHAピン留め必須
- **参考**: [Aqua Security](https://www.aquasec.com/blog/trivy-supply-chain-attack-what-you-need-to-know/), [CrowdStrike](https://www.crowdstrike.com/en-us/blog/from-scanner-to-stealer-inside-the-trivy-action-supply-chain-compromise/), [Microsoft](https://www.microsoft.com/en-us/security/blog/2026/03/24/detecting-investigating-defending-against-trivy-supply-chain-compromise/)

---

### LiteLLM PyPI侵害 - TeamPCP Phase 09 (2026年3月24日)

- **CVE**: N/A
- **対象**: `litellm@1.82.7`, `litellm@1.82.8` (PyPI)
- **攻撃ベクトル**: **Trivy侵害からのカスケード攻撃** (CI/CDパイプライン経由のPyPIトークン窃取)
- **概要**: LiteLLMのCI/CDパイプラインが侵害済みのTrivy Actionを使用していたため、`PYPI_PUBLISH` トークンが攻撃者のサーバーに送信された。攻撃者は正規の公開権限を取得し、正規のCI/CDを迂回してPyPIに直接悪意あるバージョンを公開。Trivy・KICS・LiteLLMの攻撃すべてに同一のRSA公開鍵が使用されており、TeamPCPによる一連のキャンペーン(Phase 09)と帰属される。
- **ペイロード (3段階)**:
  1. **情報収集**: SSH秘密鍵、AWS/GCP/Azure認証情報、Kubernetesトークン、暗号通貨ウォレット、環境変数、CI/CD設定ファイル等を体系的に収集
  2. **暗号化・持ち出し**: AES-256-CBC + 4096ビットRSA公開鍵でハイブリッド暗号化し、偽ドメイン `models.litellm.cloud` へPOST送信
  3. **永続化・横展開**: `~/.config/sysmon/sysmon.py` にバックドア設置、systemdサービスで5分ごとにC2をポーリング。Kubernetesサービスアカウントトークンが存在する場合、クラスタ全ノードに特権Podをデプロイしバックドアをインストール (**CanisterWorm** - ICP(Internet Computer Protocol)をC2チャネルに使用)
- **v1.82.8の特筆事項**: `litellm_init.pth` をsite-packagesに配置し、**Pythonインタプリタ起動時に自動実行**。`import litellm` すら不要で発動。pipのハッシュ検証はwheelのRECORDファイルに正しく宣言されていたため通過した。
- **影響**: 公開から隔離まで**わずか46分間**で約47,000DL（v1.82.8: 32,464、v1.82.7: 14,532）。LiteLLMは約340万DL/日。PyPI上でlitellmに依存する2,337パッケージのうち88%(2,054)が侵害バージョンを許容するバージョン指定だった。DSPy, MLflow, OpenHands, CrewAI等の主要下流プロジェクトが数時間以内にセキュリティPRを発行。
- **発見**: FutureSearch社のエンジニアがCursor MCPプラグインの推移的依存関係としてインストールされた際に、マルウェアのフォークボム的バグ（指数関数的プロセス生成）によりマシンがクラッシュし発覚。**マルウェア自体のバグが早期発見につながった**。
- **教訓**:
  - **Trivy → LiteLLM**: セキュリティスキャナの侵害が下流プロジェクトのCI/CDトークン窃取に連鎖する実例
  - `.pth` ファイルによるPython起動時の自動実行は `import` 不要で検出困難
  - **Trusted Publishing (OIDC)** への移行で長期トークンのリスクを排除すべき
  - Kubernetesクラスタへの横展開能力を持つマルウェアが登場 — クラウドインフラ全体が攻撃範囲に
  - ブロックチェーン(ICP)をC2に使用する手法はテイクダウンが困難
- **参考**: [LiteLLM公式](https://docs.litellm.ai/blog/security-update-march-2026), [Snyk](https://snyk.io/articles/poisoned-security-scanner-backdooring-litellm/), [FutureSearch](https://futuresearch.ai/blog/litellm-pypi-supply-chain-attack/), [The Register](https://www.theregister.com/2026/03/24/trivy_compromise_litellm/)

---

### GlassWorm Open VSXキャンペーン (2026年1月〜3月)

- **対象**: 72以上のOpen VSX拡張機能
- **攻撃ベクトル**: トランジティブ依存関係悪用、Unicode不可視文字によるペイロード隠蔽
- **概要**: 信頼を獲得した拡張機能が更新で悪意ある依存拡張機能を追加。SolanaトランザクションをデッドドロップリゾルバとしてC2サーバーを取得。151のGitHubリポジトリにUnicode不可視文字でペイロードを注入。
- **影響**: 900万以上のインストール
- **教訓**:
  - 拡張機能の依存関係チェーンが攻撃面になる
  - ブロックチェーンをC2インフラに使用する新手法
- **参考**: [The Hacker News](https://thehackernews.com/2026/03/glassworm-supply-chain-attack-abuses-72.html)

---

### SANDWORM_MODE npm ワーム (2026年2月20日公開)

- **対象**: 19のタイポスクワッティングパッケージ
- **攻撃ベクトル**: タイポスクワッティング + 自己増殖 + **AIツールチェーン汚染**
- **概要**: macOS/Linux/Windows全対応。ステージ1でクレデンシャル/暗号鍵窃取、ステージ2(48時間+ランダムジッター後)でパスワードマネージャーからの窃取、ワーム拡散、**MCPサーバー注入によるAIコーディングアシスタントへのプロンプトインジェクション**。
- **教訓**:
  - **AIコーディングアシスタントが新たな攻撃面に**
  - MCPサーバー注入によりAIエージェントを操作してクレデンシャルを流出させる手法が登場
  - 2段階の遅延実行で検知を回避
- **参考**: [Kodem](https://www.kodemsecurity.com/resources/sandworm-mode-a-new-shai-hulud-style-npm-worm-threatening-developer-ai-toolchain-security)

---

### AI IDE フォーク (Cursor/Windsurf) の Open VSX 脆弱性 (2026年1月)

- **対象**: Cursor, Windsurf等のAI搭載VS Codeフォーク
- **攻撃ベクトル**: 名前空間の不整合悪用
- **概要**: IDEが推奨する拡張機能がOpen VSXに存在しないため、攻撃者が同名の悪意ある拡張機能を登録可能。
- **教訓**: VS Codeフォークが異なるマーケットプレイス(Open VSX)を使用する際の名前空間リスク
- **参考**: [The Hacker News](https://thehackernews.com/2026/01/vs-code-forks-recommend-missing.html)

---

## 2025年

### chalk / debug 大規模侵害 - September 8 Incident (2025年9月8日)

- **対象**: `chalk`, `debug`, `ansi-styles`, `strip-ansi`, `supports-color` 他計18パッケージ
- **攻撃ベクトル**: メンテナアカウント乗っ取り (フィッシング)
- **概要**: 偽ドメイン `npmjs.help` からの2FAリセットフィッシングメールでメンテナアカウントを乗っ取り。週間20億以上DLのパッケージが約2時間侵害。暗号通貨ウォレットハイジャックコード(MetaMask等のwindow.ethereumをフック)が注入された。
- **影響**: 実被害額は約5セントのETHと約20USD。約16分で悪意あるコード注入、約2時間後にコミュニティとnpmチームが検知・削除。
- **教訓**:
  - 単一メンテナの侵害が巨大な爆発半径を持つ
  - TOTP方式の2FAはフィッシングに対して万全ではない
  - スコープ付きパッケージ(@org/package)の使用推奨
- **参考**: [Semgrep](https://semgrep.dev/blog/2025/chalk-debug-and-color-on-npm-compromised-in-new-supply-chain-attack/), [Vercel](https://vercel.com/blog/critical-npm-supply-chain-attack-response-september-8-2025)

---

### Shai-Hulud npm ワーム (第1波: 2025年9月, 第2波: 2025年11月)

- **対象**: 796ユニークパッケージ (週間2,000万DL以上)
- **攻撃ベクトル**: フィッシング → npmトークン窃取 → 自己増殖型ワーム
- **概要**: npmエコシステム史上初の自己複製型ワーム。窃取したnpmトークンで被害者の他パッケージにpreinstallスクリプトを注入し、自動的に最大100パッケージに拡散。Zapier, ENS Domains, PostHog, Postman等の著名パッケージが含まれた。`setup_bun.js` と `bun_environment.js` を注入。25,000以上のGitHubリポジトリ、350以上のユーザーアカウントに影響。
- **教訓**:
  - npmトークンの有効期限設定とスコープ制限が必須
  - preinstallスクリプトの自動実行が爆発的拡散を可能にした
  - 窃取データをGitHubの公開リポジトリにコミットして流出
- **参考**: [Unit42](https://unit42.paloaltonetworks.com/npm-supply-chain-attack/), [CISA](https://www.cisa.gov/news-events/alerts/2025/09/23/widespread-supply-chain-compromise-impacting-npm-ecosystem), [Microsoft](https://www.microsoft.com/en-us/security/blog/2025/12/09/shai-hulud-2-0-guidance-for-detecting-investigating-and-defending-against-the-supply-chain-attack/)

---

### UNC6426 nx npm パッケージ侵害 (2025年8月)

- **対象**: `nx` (Nrwlのモノレポツール)
- **攻撃ベクトル**: `pull_request_target` ワークフローの脆弱性 (Pwn Request攻撃)
- **概要**: 悪意あるバージョンがpostinstallでQUIETVAULT(クレデンシャルスティーラー)を実行。AWS管理者権限が72時間以内に奪取され、S3バケットアクセス、EC2/RDSインスタンス停止、内部GitHubリポジトリの公開化が行われた。
- **教訓**: `pull_request_target` ワークフローは外部PRからのコード実行リスクがある
- **参考**: [The Hacker News](https://thehackernews.com/2026/03/unc6426-exploits-nx-npm-supply-chain.html)

---

### Ethcode VS Code拡張機能 悪意あるPR攻撃 (2025年6月17日)

- **対象**: Ethcode VS Code拡張機能 (約6,000インストール)
- **攻撃ベクトル**: 悪意あるプルリクエスト
- **概要**: 43コミット、約4,000行の変更に2行の悪意あるコードを紛れ込ませたPR。偽依存パッケージ `keythereum-utils` がPowerShellでマルウェアをダウンロード。
- **教訓**: PRレビューで新しい依存関係の追加に特に注意。使い捨てアカウントからの大規模PRは警戒信号。
- **参考**: [The Hacker News](https://thehackernews.com/2025/07/malicious-pull-request-infects-6000.html)

---

### tj-actions/changed-files 侵害 (2025年3月14日-15日)

- **CVE**: CVE-2025-30066, CVE-2025-30154 (reviewdog)
- **対象**: `tj-actions/changed-files`, `reviewdog/action-setup`
- **攻撃ベクトル**: GitHub Actionバージョンタグ改ざん (カスケード型)
- **概要**: reviewdog/action-setupの侵害が起点。攻撃者が自動招待プロセスを悪用してreviewdogメンテナーチームに参加→書き込み権限取得→悪意あるコミットをプッシュ。その後、tj-actions-botのPATを窃取し、全バージョンタグを悪意あるコミットに書き換え。CI/CDシークレット(PAT、npmトークン、RSA秘密鍵等)がワークフローログに露出。当初のターゲットはCoinbaseのagentkit OSSプロジェクト。
- **影響**: 23,000以上のリポジトリ、実際に218リポジトリがシークレットを漏洩。CISAがKEVカタログに追加。
- **発見**: StepSecurity Harden-Runnerが予期しない外部通信を検知
- **教訓**:
  - GitHub Actionsはコミットハッシュでピン留め（タグは書き換え可能）
  - `GITHUB_TOKEN` のスコープを最小限に
  - サードパーティActionsの連鎖的な依存がカスケード侵害を引き起こす
- **参考**: [Wiz](https://www.wiz.io/blog/github-action-tj-actions-changed-files-supply-chain-attack-cve-2025-30066), [CISA](https://www.cisa.gov/news-events/alerts/2025/03/18/supply-chain-compromise-third-party-tj-actionschanged-files-cve-2025-30066-and-reviewdogaction), [Unit42](https://unit42.paloaltonetworks.com/github-actions-supply-chain-attack/)

---

## 2024年

### Ultralytics PyPI サプライチェーン攻撃 (2024年12月)

- **対象**: Ultralytics (AI/コンピュータビジョンライブラリ)
- **攻撃ベクトル**: GitHub Actions `pull_request_target` テンプレートインジェクション
- **概要**: gitブランチ名を操作してPyPI APIトークンを窃取。暗号通貨マイナー(XMRig)を含む悪意あるバージョン(8.3.41, 8.3.42, 8.3.45, 8.3.46)がPyPIに公開された。
- **教訓**: CI/CDワークフローでのユーザー制御入力の検証が必須
- **参考**: [PyPI Blog](https://blog.pypi.org/posts/2024-12-11-ultralytics-attack-analysis/)

---

### npm レジストリ トークンファーミング (2024年10月)

- **対象**: 15万以上の偽パッケージ
- **攻撃ベクトル**: レジストリスパム (tea.xyzプロトコル悪用)
- **概要**: ブロックチェーンベースのインセンティブシステム(tea.xyz)を悪用し、15万以上の空パッケージがnpmレジストリに投入された。
- **教訓**: ブロックチェーンインセンティブがスパムの動機付けになり得る。パッケージ信頼性の自動評価が必要。
- **参考**: [DarkReading](https://www.darkreading.com/application-security/150000-packages-flood-npm-registry-token-farming)

---

### @0xengine/xmlrpc 長期潜伏型攻撃 (2023年10月〜2024年11月)

- **対象**: `@0xengine/xmlrpc`
- **攻撃ベクトル**: 悪意あるメンテナ (正規→マルウェア化)
- **概要**: 正規パッケージとして1年以上運用された後、後のバージョンでクリプトマイニングとデータ窃取のコードが注入された。
- **教訓**: パッケージが最初は正規でも後に悪意化する可能性がある。継続的監視が重要。

---

### XZ Utils バックドア (CVE-2024-3094) - 2024年3月発見

- **CVE**: CVE-2024-3094
- **対象**: xz-utils / liblzma
- **攻撃ベクトル**: ソーシャルエンジニアリングによる長期的信頼獲得 → メンテナ権限取得 → リリースアーティファクトへのバックドア注入
- **概要**: 「Jia Tan」が2年間をかけてメンテナ権限を獲得し、SSHデーモン経由でRCEを可能にするバックドアを埋め込んだ。リリースアーティファクト（ソースコードではなく）に注入されたため、ソースコードレビューだけでは発見困難。
- **影響**: Debian unstable、Fedora Rawhide等の不安定版に到達。安定版への到達前に発見。
- **発見**: Andres FreundがSSHログインの遅延を偶然調査して発見
- **教訓**:
  - リリースアーティファクトとソースコードの一致検証が必須
  - 単一メンテナへの依存が致命的リスク
  - 国家支援型攻撃者は年単位で忍耐強く攻撃を準備する
- **参考**: [Penligent](https://www.penligent.ai/hackinglabs/xz-utils-cve-reality-check-cve-2024-3094-the-liblzma-backdoor-and-why-your-build-pipeline-was-the-real-target/)

---

## VS Code / IDE拡張機能関連 (継続的)

| 時期 | 事件 | 概要 |
|------|------|------|
| 2025年10月 | 100以上のVS Code拡張機能リスク露出 | PATトークン漏洩により15万インストールにマルウェア配布可能な状態 |
| 2025年11月 | prettier-vscode-plus | 偽Prettier拡張機能。Anivia loader + OctoRAT。4時間以内に削除 |
| 2025年2月-12月 | 19 VS Code拡張機能キャンペーン | 依存関係フォルダ内にマルウェアを隠蔽 |
| 2026年1月-3月 | GlassWorm | 72以上のOpen VSX拡張機能。Solanaブロックチェーン利用のC2 |

---

## MCP (Model Context Protocol) 関連 (新興)

| 時期 | 事件 | 概要 |
|------|------|------|
| 2025年 | Postmark MCP侵害 | バックドアによりMCPサーバー経由で全送信メールが攻撃者にBCC |
| 2025年5月 | GitHub MCP統合悪用 | パブリックリポの悪意あるIssueでAIエージェントをハイジャック |
| 2026年2月 | SANDWORM_MODE | MCPサーバー注入によるAIコーディングアシスタントへのプロンプトインジェクション |

MCPサーバーの43%にOAuth認証の脆弱性、43%にコマンドインジェクション脆弱性が報告されている。2025年だけで13,000以上のMCPサーバーがGitHubに登場。
