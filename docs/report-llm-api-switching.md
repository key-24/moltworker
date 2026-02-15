# LLM APIプロバイダ切り替えに関する調査レポート

> **作成日:** 2026-02-12
> **対象プロジェクト:** OpenClaw on Cloudflare Workers (Clawdbot)

---

## 概要

本レポートでは、以下の3点について調査・回答する。

1. Anthropic API以外のプロバイダ（Gemini API等）への切り替え可否
2. Anthropicプロプラン（アカウントログイン）での利用可否
3. アカウントログイン vs APIキーのセキュリティ比較

---

## 1. 他のAPIキーへの切り替えは可能か？

### 結論：部分的に可能

### 現在サポートされているプロバイダ

`src/gateway/env.ts` および `wrangler.jsonc` の設定から確認した、対応済みプロバイダは以下の通り。

| 方法 | 対応プロバイダ | 設定方法 |
|------|------------|---------|
| 直接APIキー | Anthropic, OpenAI | `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` |
| Cloudflare AI Gateway経由 | Anthropic, OpenAI, Groq, Workers AI | `CLOUDFLARE_AI_GATEWAY_API_KEY` + Gateway設定 |

### 切り替え手順（OpenAIの場合）

```bash
# OpenAI APIキーを設定
npx wrangler secret put OPENAI_API_KEY

# 再デプロイ
npm run deploy
```

### Gemini APIについて

> [!WARNING]
> Gemini APIは現時点でこのプロジェクトに直接サポートされていない。

利用するには以下のいずれかの対応が必要：

1. **Cloudflare AI Gateway経由** — AI GatewayがGoogle AI（Gemini）をサポートしていれば、`CF_AI_GATEWAY_MODEL` に `google-ai/gemini-2.0-flash` のような形式で設定できる可能性がある。ただし、OpenClaw本体がGeminiのレスポンス形式に対応しているかは別途検証が必要。
2. **OpenAI互換API** — GeminiがOpenAI互換エンドポイントを提供していれば、`OPENAI_API_KEY` + カスタムベースURLで動作する可能性がある。

---

## 2. Anthropicプロプランのアカウントログインで利用できるか？

### 結論：利用可能（OAuthトークン経由）

> [!IMPORTANT]
> 2026-02-15更新: 詳細調査の結果、OpenClawはClaude Pro/MaxプランのOAuthトークン認証に対応していることが判明。以下の手順で設定可能。

### セットアップ手順

1. **Claude Code CLIをインストール**（ローカルPCで実行）
   ```bash
   winget install Anthropic.ClaudeCode
   ```

2. **OAuthトークンを取得**
   ```bash
   claude setup-token
   # → ブラウザが開くのでProプランのアカウントでログイン
   # → ターミナルにトークン（sk-ant-oat01-...）が表示される
   ```

3. **Wrangler secretとして設定**
   ```bash
   npx wrangler secret put ANTHROPIC_OAUTH_TOKEN
   # → 取得したトークンを貼り付け
   ```

4. **再デプロイ**
   ```bash
   npm run deploy
   ```

5. **（任意）APIキーを削除**
   ```bash
   npx wrangler secret delete ANTHROPIC_API_KEY
   ```

### 仕組み

- OAuthトークンの取得はローカルPCで1回だけ行う（ブラウザログイン）
- 取得したトークンはWrangler secretとしてCloudflareに保存される
- コンテナ起動時、`start-openclaw.sh`が`--auth-choice anthropic-oauth`でonboardを実行
- APIキーの従量課金が不要になり、Proプラン（$20/月）の定額で利用可能

### Claude Code CLIとの比較

| 項目 | Claude Code CLI | OpenClaw（本プロジェクト） |
|------|---------------|------------------------|
| 実行環境 | ローカルPC | Cloudflare Workers / Sandbox |
| 認証方式 | OAuth（ブラウザログイン）or APIキー | OAuthトークン or APIキー |
| プロプラン利用 | ✅ 可能 | ✅ 可能（OAuthトークン経由） |
| 課金先 | Anthropicアカウント | Proプラン定額 or APIキー従量課金 |

---

## 3. アカウントログイン vs APIキーのセキュリティ比較

> [!NOTE]
> 本セクションは②が不可のため参考情報として記載。

### APIキー認証

| 項目 | 内容 |
|------|------|
| キー漏洩 | 漏洩すると誰でもAPI呼び出し可能 |
| スコープ | API呼び出しのみに限定（アカウント設定変更等は不可） |
| 無効化 | キーの無効化・ローテーションが容易 |
| 利用量制限 | レート制限や使用上限を設定可能 |

### アカウントログイン（OAuth）

| 項目 | 内容 |
|------|------|
| アクセス範囲 | アカウント全体への権限を持つ可能性がある |
| セッション管理 | トークンの有効期限管理が必要 |
| 2FA保護 | 二要素認証でトークン漏洩時の被害を限定可能 |
| 同意の明示性 | OAuthフローで付与権限が明示される |

### 総合評価

| 観点 | APIキー | OAuthログイン |
|------|---------|-------------|
| セキュリティ | 🟢 スコープ限定で制御しやすい | 🟡 広範な権限の可能性 |
| 運用 | 🟡 キー管理が必要 | 🟢 セッションベースで管理不要 |
| 漏洩時リスク | 🟢 影響限定的、即座に無効化可能 | 🟡 アカウント全体に影響の可能性 |

> [!IMPORTANT]
> サーバーサイドアプリケーションでは **APIキー認証の方がセキュリティ的に適切** である。スコープが限定されており、漏洩時の対処も容易。アカウントログインはローカルCLIツール（Claude Code等）での利用に適している。

---

## 参考：確認したソースファイル

- `src/gateway/env.ts` — 環境変数の構築ロジック
- `src/gateway/process.ts` — ゲートウェイプロセスの起動ロジック
- `wrangler.jsonc` — Cloudflare Workers設定（シークレット一覧）
- `.dev.vars.example` — 開発用環境変数テンプレート
- `README.md` — プロジェクトドキュメント（AI Gateway・シークレットリファレンス）
