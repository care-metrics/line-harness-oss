# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

LINE公式アカウント向けOSSのCRM/マーケティング自動化プラットフォーム。Cloudflare無料枠で動作し、L社・U社の代替として設計されている。

## 技術スタック

- **Backend:** Cloudflare Workers + Hono 4.x
- **Database:** Cloudflare D1 (SQLite) — 42テーブル
- **Frontend (admin):** Next.js 15 (App Router, static export)
- **Frontend (LIFF):** Vite — Workerの静的アセットとして配信
- **SDK:** `@line-harness/sdk` (ESM + CJS, tsup)
- **MCP Server:** `@line-harness/mcp-server` (Claude統合用)
- **言語:** TypeScript (strict mode), ES2022ターゲット
- **パッケージマネージャ:** pnpm 9.x (workspaces)
- **Node:** >=20

## コマンド

```bash
# 開発
pnpm dev:worker          # Worker開発サーバー (port 8787)
pnpm dev:web             # Next.js開発サーバー (port 3001)

# ビルド
pnpm build               # 全パッケージビルド（ビルド順: shared → line-sdk → db → worker/web）
pnpm --filter worker build
pnpm --filter web build
pnpm --filter sdk build  # tsup (ESM + CJS dual format)

# デプロイ
pnpm deploy:worker       # Cloudflareへデプロイ

# データベース
pnpm db:migrate          # D1本番にスキーマ適用
pnpm db:migrate:local    # D1ローカルにスキーマ適用

# テスト
cd packages/sdk && pnpm vitest       # SDKテスト実行
cd packages/sdk && pnpm vitest run   # CI向き（watch無し）
cd packages/sdk && pnpm vitest <ファイル名>  # 単一テスト実行
```

## モノレポ構成

```
apps/
  worker/    → Cloudflare Workers (Hono) — APIサーバー + LIFF配信
  web/       → Next.js 15 管理画面 (static export)
packages/
  db/        → D1クエリヘルパー + schema.sql（ビルド不要、ソース直接参照）
  sdk/       → 公開SDK (@line-harness/sdk)
  shared/    → 共有型・定数
  line-sdk/  → LINE Messaging API型付きラッパー
  mcp-server/→ Claude MCP Server
  create-line-harness/ → セットアップCLI
  plugin-template/     → プラグインテンプレート
```

内部パッケージ間の依存は `workspace:*` で管理。

## アーキテクチャ

### Worker (API Server)

`apps/worker/src/index.ts` がエントリポイント。Honoルーターに27個のルートモジュール（100+エンドポイント）をマウント。CORS・認証・レートリミットはミドルウェアで処理。

**サービス層** (`apps/worker/src/services/`): Cronトリガー（5分間隔）で実行されるバックグラウンド処理。
- ステップ配信、ブロードキャスト、リマインダー配信
- BANモニタリング、トークンリフレッシュ
- セグメントクエリ/送信、自動追跡
- event-bus.ts がイベントオーケストレーションの中核

### DB層

`packages/db/src/` に25個のクエリモジュール。各モジュールがインターフェース定義 + D1ラッパー関数をエクスポート。ORMは不使用、生SQLクエリ。スキーマは `packages/db/schema.sql` で管理。

### 認証/マルチアカウント

複数LINE公式アカウントをDB内のトークンで管理。APIキー認証 + アカウントID指定でマルチテナント動作。

### 環境変数

`.env.example` 参照。主要なシークレット:
- `API_KEY` — API認証キー
- `LINE_CHANNEL_*` — LINEチャネル認証情報
- `WEBHOOK_SECRET` — Webhook検証用

## CI/CD

`.github/workflows/deploy-worker.yml` — main pushまたは手動トリガーでWorkerをデプロイ。ビルド順序: shared → line-sdk → db → worker。

## OSS同期

`scripts/sync-oss.sh` でプライベートリポからOSSへ同期。シークレットのサニタイズ・リーク検証を含む。詳細は `docs/OSS-SYNC-CHARTER.md` を参照。
