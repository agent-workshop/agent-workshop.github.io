---
layout: post
title: "BunのHTTPサーバーでAI APIプロキシを構築する"
date: 2026-05-17 09:00:00 +0900
categories: [Bun, TypeScript]
tags: [Bun, TypeScript, API, Claude, 生成AI]
description: BunのHTTPサーバーでAI APIプロキシを構築するの実装方法を解説します。
---
# BunのHTTPサーバーでAI APIプロキシを構築する

## 1. 導入

AI API（OpenAI、Anthropicなど）を直接クライアントから呼び出すと、APIキーの漏洩リスクやレート制限の管理が課題になります。また、複数のAIサービスを統合して利用する場合、各APIのエンドポイントや認証方式の違いを吸収する仕組みが必要です。

そこで本記事では、高速ランタイム「Bun」を使って、AI APIのプロキシサーバーを構築する方法を紹介します。BunのHTTPサーバーは標準ライブラリのみで動作し、Node.jsと比較して起動が速く、依存関係も最小限に抑えられます。これにより、以下のメリットが得られます。

- APIキーをサーバー側で一元管理し、クライアントに露出させない
- レート制限やリクエストのログをサーバーで集約できる
- 複数のAI APIのエンドポイントを統一したインターフェースで利用できる

## 2. 前提条件

- **Bun**（v1.0以上）がインストールされていること
  - インストール: `curl -fsSL https://bun.sh/install | bash`
- TypeScriptの基本的な知識があること
- OpenAIまたはAnthropicのAPIキーを取得していること（テスト用）

## 3. 実装手順

### ステップ1: プロジェクトの初期化

```bash
mkdir ai-proxy
cd ai-proxy
bun init -y
```

`bun init -y`で`package.json`と`tsconfig.json`が生成されます。

### ステップ2: 環境変数の設定

`.env`ファイルを作成し、APIキーを管理します。

```bash
touch .env
```

`.env`の内容:

```
OPENAI_API_KEY=sk-your-openai-key
ANTHROPIC_API_KEY=sk-ant-your-anthropic-key
PORT=3000
```

### ステップ3: プロキシサーバーの実装

`index.ts`を作成し、BunのHTTPサーバーを実装します。

```typescript
import { serve } from "bun";

const PORT = parseInt(process.env.PORT || "3000", 10);
const OPENAI_API_KEY = process.env.OPENAI_API_KEY || "";
const ANTHROPIC_API_KEY = process.env.ANTHROPIC_API_KEY || "";

// リクエストログの出力
function logRequest(method: string, path: string, status: number, duration: number) {
  console.log(`[${new Date().toISOString()}] ${method} ${path} -> ${status} (${duration}ms)`);
}

serve({
  port: PORT,
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const startTime = Date.now();

    // CORSヘッダーの設定（開発用）
    const corsHeaders = {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Methods": "POST, OPTIONS",
      "Access-Control-Allow-Headers": "Content-Type, Authorization",
    };

    // Preflightリクエストの処理
    if (request.method === "OPTIONS") {
      return new Response(null, { headers: corsHeaders });
    }

    // POSTリクエストのみ許可
    if (request.method !== "POST") {
      return new Response(JSON.stringify({ error: "Method not allowed" }), {
        status: 405,
        headers: { "Content-Type": "application/json", ...corsHeaders },
      });
    }

    try {
      const body = await request.json();
      const { model, messages, max_tokens, temperature } = body;

      // モデル名からプロバイダーを判定
      let targetUrl: string;
      let apiKey: string;

      if (model.startsWith("gpt-")) {
        // OpenAI
        targetUrl = "https://api.openai.com/v1/chat/completions";
        apiKey = OPENAI_API_KEY;
      } else if (model.startsWith("claude-")) {
        // Anthropic
        targetUrl = "https://api.anthropic.com/v1/messages";
        apiKey = ANTHROPIC_API_KEY;
      } else {
        return new Response(JSON.stringify({ error: "Unsupported model" }), {
          status: 400,
          headers: { "Content-Type": "application/json", ...corsHeaders },
        });
      }

      // リクエスト転送
      const response = await fetch(targetUrl, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${apiKey}`,
        },
        body: JSON.stringify({
          model,
          messages,
          max_tokens: max_tokens || 1024,
          temperature: temperature || 0.7,
        }),
      });

      const data = await response.json();
      const duration = Date.now() - startTime;
      logRequest(request.method, url.pathname, response.status, duration);

      return new Response(JSON.stringify(data), {
        status: response.status,
        headers: { "Content-Type": "application/json", ...corsHeaders },
      });
    } catch (error) {
      console.error("Proxy error:", error);
      return new Response(JSON.stringify({ error: "Internal server error" }), {
        status: 500,
        headers: { "Content-Type": "application/json", ...corsHeaders },
      });
    }
  },
});

console.log(`AI Proxy server running on http://localhost:${PORT}`);
```

### ステップ4: 起動

```bash
bun run index.ts
```

サーバーが起動したら、以下のようなリクエストを送信して動作確認します。

```bash
curl -X POST http://localhost:3000 \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-3.5-turbo",
    "messages": [{"role": "user", "content": "Hello, AI!"}]
  }'
```

## 4. コードサンプル

### サンプル1: クライアントからの呼び出し（フロントエンド想定）

```typescript
// client.ts
async function callAIProxy(model: string, message: string) {
  const response = await fetch("http://localhost:3000", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model,
      messages: [{ role: "user", content: message }],
      max_tokens: 500,
      temperature: 0.5,
    }),
  });

  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }

  return await response.json();
}

// 使用例
callAIProxy("gpt-3.5-turbo", "What is Bun?")
  .then((data) => console.log(data.choices[0].message.content))
  .catch(console.error);
```

### サンプル2: レート制限を追加したプロキシ

```typescript
// rate-limited-proxy.ts
import { serve } from "bun";

const RATE_LIMIT = 10; // 1分間あたりの最大リクエスト数
const WINDOW_MS = 60 * 1000; // 1分

const requestCounts = new Map<string, { count: number; resetTime: number }>();

function checkRateLimit(ip: string): boolean {
  const now = Date.now();
  const record = requestCounts.get(ip);

  if (!record || now > record.resetTime) {
    // 新しいウィンドウを開始
    requestCounts.set(ip, { count: 1, resetTime: now + WINDOW_MS });
    return true;
  }

  if (record.count >= RATE_LIMIT) {
    return false; // レート制限超過
  }

  record.count++;
  return true;
}

serve({
  port: 3001,
  async fetch(request: Request): Promise<Response> {
    const ip = request.headers.get("x-forwarded-for") || "unknown";

    if (!checkRateLimit(ip)) {
      return new Response(
        JSON.stringify({ error: "Rate limit exceeded. Try again later." }),
        { status: 429, headers: { "Content-Type": "application/json" } }
      );
    }

    // ここにプロキシ処理（サンプル1の内容）を記述
    // ...
    return new Response("OK");
  },
});

console.log("Rate-limited proxy running on http://localhost:3001");
```

## 5. ハマりやすいポイント

### CORSエラー
フロントエンドから直接呼び出す場合、ブラウザのCORSポリシーに引っかかります。上記のサンプルでは`Access-Control-Allow-Origin: *`を設定していますが、本番環境では適切なオリジンを指定してください。

### ストリーミングレスポンスの非対応
上記のプロキシは、OpenAIのストリーミングモード（`stream: true`）に対応していません。ストリーミングを実装する場合は、`ReadableStream`を使用してレスポンスをパイプする必要があります。

### 環境変数の読み込み
Bunはデフォルトで`.env`ファイルを自動読み込みしません。`bun run`で起動するか、`dotenv`パッケージを使用する必要があります。上記のコードでは、`bun run index.ts`で実行すれば自動的に`.env`が読み込まれます。

### エラーハンドリングの不足
本番運用では、APIキーの有効期限切れやネットワークエラーなど、より詳細なエラーハンドリングが必要です。上記のサンプルでは簡易的な実装に留めています。

## 6. まとめ

BunのHTTPサーバーを使うことで、最小限のコードでAI APIプロキシを構築できました。以下のポイントが特に有用です。

- **APIキーの隠蔽**: クライアントにキーを露出せずに済む
- **統一インターフェース**: 複数のAIプロバイダーを透過的に扱える
- **高速・軽量**: Bunのランタイムは起動が高速で、メモリ使用量も少ない

本記事のコードをベースに、認証機能やキャッシュ、ログ収集などを追加することで、より実践的なプロキシサーバーに拡張できます。ぜひ自身のユースケースに合わせてカスタマイズしてみてください。

---

**あわせて読みたい**

AIエージェント開発のさらに深い実践知については、こちらのnote記事で詳しく解説しています。実務での自動化やチーム開発のヒントになれば幸いです。

[AIエージェント開発で踏んだ5つの地雷（コード付き）](https://note.com/agent_workshop/n/n6481c8a63571)