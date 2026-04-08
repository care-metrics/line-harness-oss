# Cloudflare 初回デプロイ設計

## 概要

LINE Harness OSSをCloudflareに初回デプロイする。セットアップCLI（`pnpm deploy:setup`）を使用し、Worker + D1 + R2 + Pagesの全スタックを一括セットアップする。カスタムドメインは使用せず、`*.workers.dev` / `*.pages.dev` で運用する。CI/CDは後回しとし、手動デプロイで動作確認を優先する。

## 前提条件

- Cloudflareアカウント作成済み、`wrangler login` 認証済み
- LINE側クレデンシャル取得済み（Channel Access Token, Channel Secret, Login Channel ID/Secret, LIFF ID）
- Node.js >= 20, pnpm 9.x インストール済み

## デプロイ構成

```
Cloudflare Workers          Cloudflare Pages
┌─────────────────────┐    ┌──────────────────┐
│  line-harness        │    │  line-harness-    │
│  (Hono API + LIFF)  │    │  admin-xxxx       │
│                      │    │  (Next.js静的)    │
│  - API endpoints     │    │                  │
│  - Webhook受信       │    │  - 管理画面      │
│  - LIFF静的配信      │    │                  │
│  - Cron (5分間隔)    │    │                  │
└──────────┬───────────┘    └──────────────────┘
           │
     ┌─────┴──────┐
     │            │
┌────▼─────┐ ┌───▼──────┐
│ D1 (SQLite)│ │ R2       │
│ 42テーブル  │ │ 画像保存  │
└────────────┘ └──────────┘
```

## 手順

### フェーズ1: 事前準備

1. **R2課金の有効化**
   - Cloudflareダッシュボード > R2 Object Storage に移動
   - プランを有効化する（無料枠: 10GB/月ストレージ、Class A 100万リクエスト/月）
   - CLIからも確認プロンプトが出る

2. **LINE クレデンシャルの確認**
   - Messaging API Channel ID（数字）
   - Channel Secret（LINE Official Account Managerから）
   - Channel Access Token（LINE Developersコンソールで発行）
   - LINE Login Channel ID（Messaging APIとは別チャネル）
   - LIFF ID（形式: `channelId-randomString`、公開状態であること）

### フェーズ2: セットアップCLI実行

```bash
pnpm deploy:setup
```

CLIが以下を自動実行する：

| ステップ | 内容 | 自動/対話 |
|---------|------|----------|
| 1 | Cloudflare認証確認 | 自動 |
| 2 | アカウントID取得 | 自動（複数アカウント時は選択） |
| 3 | R2課金確認 | 対話（Enter押下） |
| 4 | プロジェクト名入力 | 対話（デフォルト: `line-harness`） |
| 5 | LINE認証情報入力 | 対話（4項目） |
| 6 | LIFF ID入力 | 対話 |
| 7 | APIキー生成 | 自動（64文字ランダム） |
| 8 | D1データベース作成 | 自動 |
| 9 | スキーマ + 22マイグレーション適用 | 自動 |
| 10 | R2バケット作成 | 自動 |
| 11 | Bot Basic ID取得 | 自動（失敗しても継続） |
| 12 | Workerビルド＆デプロイ | 自動 |
| 13 | シークレット5件設定 | 自動 |
| 14 | LINEアカウント登録 | 自動（失敗しても継続） |
| 15 | Next.js管理画面ビルド＆Pagesデプロイ | 自動 |

**中断時の挙動:** ステート（`.line-harness-setup.json`）に進捗が保存され、再実行で続きから再開可能。

**生成されるリソース:**
- Worker: `https://line-harness.{subdomain}.workers.dev`
- Pages: `https://line-harness-admin-{suffix}.pages.dev`
- D1: `line-harness`（42テーブル + インデックス）
- R2: `line-harness-images`
- シークレット: `LINE_CHANNEL_ACCESS_TOKEN`, `LINE_CHANNEL_SECRET`, `LINE_LOGIN_CHANNEL_ID`, `LIFF_URL`, `API_KEY`

### フェーズ3: デプロイ後の手動設定

1. **Webhook URL設定**
   - LINE Official Account Manager > Messaging API設定
   - Webhook URL: CLIが表示する `https://{worker-url}/webhook`
   - Webhookを有効化

2. **LIFFエンドポイントURL更新**
   - LINE Developersコンソール > LIFFアプリ設定
   - エンドポイントURL: `https://{worker-url}/liff`

3. **動作確認**
   - 管理画面にアクセスし、APIキーでログイン
   - LINE公式アカウントに友だち追加して、Webhook受信を確認
   - LIFFページが正常に表示されることを確認

## 注意事項

- APIキーはCLI完了時に一度だけ表示される。必ず控えておくこと
- `.line-harness-config.json` がリポジトリルートに生成される。`.gitignore` に追加推奨
- Cron（5分間隔）はWorkerデプロイ後すぐに動き始める
- Cloudflare無料枠の主な制限: Workers 10万リクエスト/日、D1 5MBストレージ、R2 10GB/月

## スコープ外（今回は対象外）

- カスタムドメイン設定
- GitHub Actions CI/CD設定
- 本番環境のパフォーマンスチューニング
- セットアップCLIのバグ修正（SQLインジェクション等の軽微な問題は既知だが今回は対処しない）
