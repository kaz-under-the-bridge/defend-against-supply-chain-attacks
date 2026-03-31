# defend-against-supply-chain-attacks

サプライチェーン攻撃への対処手法を壁打ちして整理しておく

## 背景

- npmを中心としたサプライチェーン攻撃が激化している
- Trivyのリポジトリ単位での乗っ取りなど、対岸の火事ではない
- 既存の脆弱性アップデート（Dependabot等）だけでは防げない
- 複数企業の管理権限を保持する立場として、個人トークン流出はクリティカル

## ドキュメント構成

| ファイル | 内容 |
|---------|------|
| [incidents.md](incidents.md) | インシデントカタログ - 2024年後半〜2026年3月の主要事件を時系列で整理 |
| [attack-vectors.md](attack-vectors.md) | 攻撃ベクトル分類 - 攻撃手法の体系化と発生頻度・影響度マトリクス |
| [defense-strategies.md](defense-strategies.md) | 防御戦略 - 個人トークン保護、GitHub Actions保護、依存関係管理等 |
| [package-validation.md](package-validation.md) | パッケージ正当性検証 - Socket.dev, SBOM, lockfileレビュー, postinstall無効化 |
| [socket-dev-setup.md](socket-dev-setup.md) | Socket.dev導入・設定ガイド - org全体での必須チェック化手順 |

## 主要な知見

### 直近の重大インシデント (2024-2026)

- **Axios** (2026年3月) - 週間8,300万DLのパッケージ、メンテナアカウント乗っ取り
- **Trivy** (2026年2-3月) - セキュリティスキャナ自体がサプライチェーン攻撃の対象に
- **chalk/debug** (2025年9月) - 週間20億DL、フィッシングによるメンテナ乗っ取り
- **Shai-Hulud** (2025年9-11月) - npmエコシステム史上初の自己複製型ワーム
- **tj-actions** (2025年3月) - GitHub Actionsタグ改ざんによるカスケード侵害
- **XZ Utils** (2024年3月) - 2年間のソーシャルエンジニアリングによるバックドア

### 最優先の防御アクション

1. **Fine-grained PATへの移行** - Classic PATは1つの漏洩で全組織が危険に
2. **GitHub ActionsのSHAピン留め** - タグは書き換え可能
3. **ハードウェアセキュリティキー** - TOTPはフィッシングに脆弱
4. **Pre-commitシークレット検出** - Gitleaks等の導入
5. **GitHub Push Protectionの有効化** - プッシュ時にシークレットをブロック
