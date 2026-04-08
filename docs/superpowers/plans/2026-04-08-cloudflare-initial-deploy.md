# Cloudflare 初回デプロイ実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** LINE Harness OSSをCloudflareに初回デプロイし、Worker + D1 + R2 + Pagesの全スタックを動作可能にする

**Architecture:** セットアップCLI（`pnpm deploy:setup`）が D1データベース作成、R2バケット作成、Workerデプロイ、シークレット設定、Pages デプロイを自動実行する。CLI実行前にビルドと事前準備が必要。デプロイ後にLINE側の手動設定を行う。

**Tech Stack:** Cloudflare Workers, D1, R2, Pages, wrangler CLI, pnpm, tsup

---

### Task 1: 依存関係インストールとCLIビルド

**Files:**
- 確認: `packages/create-line-harness/package.json`
- 生成: `packages/create-line-harness/dist/index.js`

- [ ] **Step 1: 依存関係をインストール**

```bash
cd /Users/riku/Develop/line-harness-oss
pnpm install
```

Expected: `dependencies` が解決され、`node_modules/` が作成される。エラーなし。

- [ ] **Step 2: セットアップCLIをビルド**

```bash
pnpm --filter create-line-harness build
```

Expected: `packages/create-line-harness/dist/index.js` が生成される。

- [ ] **Step 3: CLIが実行可能か確認**

```bash
node packages/create-line-harness/dist/index.js --help
```

Expected: ヘルプメッセージまたはCLIが起動する（対話プロンプトが表示されればOK、Ctrl+Cで中断）。

---

### Task 2: wrangler認証の確認

- [ ] **Step 1: wranglerの認証状態を確認**

```bash
npx wrangler whoami
```

Expected: アカウント名とアカウントIDが表示される。例:
```
⛅️ wrangler
 ⬣ Getting User settings...
 👋 You are logged in with an OAuth Token, associated with the email user@example.com!
 ┌─────────────────────┬──────────────────────────────────┐
 │ Account Name        │ Account ID                       │
 ├─────────────────────┼──────────────────────────────────┤
 │ Your Account        │ abcdef1234567890abcdef1234567890 │
 └─────────────────────┴──────────────────────────────────┘
```

もし認証されていない場合は `npx wrangler login` を実行してブラウザ認証を完了する。

---

### Task 3: R2課金の有効化（手動・ダッシュボード操作）

- [ ] **Step 1: R2を有効化**

1. ブラウザで Cloudflareダッシュボード（https://dash.cloudflare.com）を開く
2. 左メニューから「R2 Object Storage」を選択
3. 「Purchase R2 Plan」または「Get Started」をクリック
4. 無料プランを選択して有効化（クレジットカード登録が必要な場合あり）

Expected: R2ダッシュボードにバケット一覧画面が表示される状態になる。

---

### Task 4: LINE クレデンシャルの準備（手動・確認作業）

- [ ] **Step 1: 以下のクレデンシャルを手元に用意**

CLIの対話プロンプトで入力するため、以下をテキストエディタ等にコピーしておく：

| 項目 | 取得場所 | 形式 |
|------|---------|------|
| Messaging API Channel ID | LINE Developers > チャネル基本設定 | 数字 |
| Channel Secret | LINE Official Account Manager > Messaging API設定 | 英数字32文字 |
| Channel Access Token | LINE Developers > Messaging API > 長期トークン発行 | 長い英数字文字列 |
| LINE Login Channel ID | LINE Developers > LINEログインチャネル > 基本設定 | 数字 |
| LIFF ID | LINE Developers > LINEログインチャネル > LIFF | `channelId-randomString` 形式 |

**注意:** LIFFアプリは「公開」状態にしておくこと（「開発中」だと動作しない）。

---

### Task 5: セットアップCLI実行

**Files:**
- 生成: `apps/worker/wrangler.toml`（CLIが一時的に書き換え、完了後に復元）
- 生成: `.line-harness-config.json`（リポジトリルート）
- 生成: `.line-harness-setup.json`（実行中のみ、完了後削除）

- [ ] **Step 1: セットアップCLIを実行**

```bash
cd /Users/riku/Develop/line-harness-oss
pnpm deploy:setup
```

対話プロンプトに以下の順で回答する：

1. **R2課金確認** → Enterを押す（Task 3で済み）
2. **プロジェクト名** → デフォルト `line-harness` のままEnter、またはカスタム名を入力
3. **Messaging API Channel ID** → 数字を入力
4. **Channel Secret** → 英数字を入力
5. **Channel Access Token** → トークンを貼り付け
6. **LINE Login Channel ID** → 数字を入力
7. **LIFF ID** → `channelId-randomString` を入力
8. **MCP設定** → 必要に応じてYes/No

Expected: 以下のリソースが作成される：
- D1データベース（スキーマ + 22マイグレーション適用済み）
- R2バケット（`{projectName}-images`）
- Worker（`https://{projectName}.{subdomain}.workers.dev`）
- シークレット5件設定済み
- Pages（`https://{projectName}-admin-{suffix}.pages.dev`）

完了画面にWorker URL、Pages URL、APIキーが表示される。

**重要: APIキーを必ず控えること。再表示されない。**

- [ ] **Step 2: エラー発生時の対処**

CLIが途中で失敗した場合、再実行すれば続きから再開される：

```bash
pnpm deploy:setup
```

ステートファイル `.line-harness-setup.json` に進捗が保存されているため、完了済みステップはスキップされる。

---

### Task 6: デプロイ結果の確認

- [ ] **Step 1: Workerの動作確認**

CLIが表示したWorker URLにアクセスする：

```bash
curl https://{your-worker-url}/
```

Expected: レスポンスが返る（200 OK またはアプリのルートレスポンス）。

- [ ] **Step 2: 管理画面の動作確認**

ブラウザでCLIが表示したPages URLにアクセスする：
- `https://{projectName}-admin-{suffix}.pages.dev`

Expected: ログイン画面が表示される。CLIで表示されたAPIキーを入力してログインできる。

- [ ] **Step 3: D1データベースの確認**

```bash
npx wrangler d1 execute {projectName} --remote --command "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name LIMIT 10;"
```

Expected: テーブル名の一覧が表示される（`auto_replies`, `broadcasts`, `conversion_events` 等）。

---

### Task 7: LINE Webhook設定（手動・ダッシュボード操作）

- [ ] **Step 1: Webhook URLを設定**

1. LINE Official Account Manager（https://manager.line.biz）を開く
2. 対象アカウントを選択
3. 「設定」>「Messaging API」を開く
4. Webhook URL に入力: `https://{your-worker-url}/webhook`
5. 「Webhookの利用」をオンにする
6. 「検証」ボタンを押す

Expected: 「成功」と表示される。

- [ ] **Step 2: 応答設定の確認**

同じMessaging API設定画面で：
- 「応答メッセージ」をオフにする（LINE Harnessが応答を制御するため）
- 「あいさつメッセージ」は任意

---

### Task 8: LIFFエンドポイントURL更新（手動・ダッシュボード操作）

- [ ] **Step 1: LIFFエンドポイントを更新**

1. LINE Developersコンソール（https://developers.line.biz）を開く
2. LINEログインチャネルを選択
3. 「LIFF」タブを開く
4. 対象LIFFアプリの「エンドポイントURL」を編集
5. URLを入力: `https://{your-worker-url}/liff`
6. 保存

Expected: エンドポイントURLが更新される。

---

### Task 9: .gitignore更新とコミット

**Files:**
- Modify: `.gitignore`

- [ ] **Step 1: .gitignoreにCLI設定ファイルを追加**

`.gitignore` の末尾に以下を追加：

```
.line-harness-config.json
.line-harness-setup.json
```

- [ ] **Step 2: 変更をコミット**

```bash
git add .gitignore
git commit -m "chore: add line-harness config files to .gitignore"
```

---

### Task 10: エンドツーエンド動作確認

- [ ] **Step 1: 友だち追加テスト**

LINE公式アカウントのQRコードまたはBot Basic IDで友だち追加する。

Expected: 管理画面の「友だち一覧」に新しい友だちが表示される。

- [ ] **Step 2: メッセージ送受信テスト**

LINEアプリから公式アカウントにテストメッセージを送信する。

Expected: 管理画面の「チャット」またはメッセージログにメッセージが記録される。

- [ ] **Step 3: LIFFアクセステスト**

LINEアプリ内でLIFF URLを開く: `https://liff.line.me/{your-liff-id}`

Expected: LIFFページ（フォーム等）が正常に表示される。

- [ ] **Step 4: Cron動作確認**

Cloudflareダッシュボード > Workers > {projectName} > ログ で、5分間隔のCron実行ログを確認する。

Expected: Cron triggerのログが表示される（初回は5分以内に実行される）。
