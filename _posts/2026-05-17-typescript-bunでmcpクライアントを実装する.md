---
layout: post
title: "TypeScript + BunでMCPクライアントを実装する"
date: 2026-05-17 09:00:00 +0900
categories: [MCP, Bun]
tags: [MCP, Bun, TypeScript, Claude, AIエージェント]
description: TypeScript + BunでMCPクライアントを実装するの実装方法を解説します。
---
# TypeScript + BunでMCPクライアントを実装する

## 1. 導入

近年、AIアシスタントやコード生成ツールの発展に伴い、外部ツールとの連携が重要になっています。MCP（Model Context Protocol）は、AIモデルが外部のツールやデータソースと安全にやり取りするためのプロトコルです。Bunは高速なJavaScriptランタイムとして注目されており、TypeScriptとの親和性も高いです。

この記事では、TypeScriptとBunを使ってMCPクライアントを実装する方法を解説します。MCPクライアントを実装することで、AIモデルからデータベース操作やファイルアクセス、外部API呼び出しなどを安全に実行できるようになります。

## 2. 前提条件

以下の環境が必要です。

- Node.js 18以上（Bunのインストールに使用）
- Bun 1.0以上
- TypeScript 5.0以上
- 基本的なTypeScript/JavaScriptの知識

### インストール

```bash
# Bunのインストール
curl -fsSL https://bun.sh/install | bash

# プロジェクト作成
mkdir mcp-client-example
cd mcp-client-example
bun init
```

## 3. 実装手順

### 3.1 依存関係のインストール

```bash
bun add @modelcontextprotocol/sdk
bun add -d @types/node typescript
```

### 3.2 TypeScriptの設定

`tsconfig.json`を作成します。

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "strict": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"]
}
```

### 3.3 基本的なMCPクライアントの実装

`src/client.ts`を作成し、MCPクライアントを実装します。

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
import { spawn } from "bun";

interface ToolDefinition {
  name: string;
  description: string;
  inputSchema: Record<string, unknown>;
}

class MCPClient {
  private client: Client;
  private transport: StdioClientTransport | null = null;

  constructor() {
    this.client = new Client(
      {
        name: "example-client",
        version: "1.0.0",
      },
      {
        capabilities: {
          tools: {},
        },
      }
    );
  }

  async connect(serverPath: string): Promise<void> {
    const serverProcess = spawn([serverPath], {
      stdin: "pipe",
      stdout: "pipe",
      stderr: "pipe",
    });

    this.transport = new StdioClientTransport({
      stdin: serverProcess.stdin,
      stdout: serverProcess.stdout,
    });

    await this.client.connect(this.transport);
    console.log("MCPサーバーに接続しました");
  }

  async listTools(): Promise<ToolDefinition[]> {
    const result = await this.client.listTools();
    return result.tools.map((tool) => ({
      name: tool.name,
      description: tool.description,
      inputSchema: tool.inputSchema,
    }));
  }

  async callTool(toolName: string, args: Record<string, unknown>): Promise<unknown> {
    const result = await this.client.callTool({
      name: toolName,
      arguments: args,
    });
    return result.content;
  }

  async disconnect(): Promise<void> {
    await this.client.close();
    if (this.transport) {
      this.transport.close();
    }
    console.log("接続を切断しました");
  }
}

export { MCPClient, ToolDefinition };
```

### 3.4 サンプルサーバーの実装

テスト用のMCPサーバーを実装します。`src/server.ts`を作成します。

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  {
    name: "example-server",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "greet",
        description: "ユーザーに挨拶を返します",
        inputSchema: {
          type: "object",
          properties: {
            name: {
              type: "string",
              description: "挨拶する相手の名前",
            },
          },
          required: ["name"],
        },
      },
      {
        name: "calculate",
        description: "簡単な計算を行います",
        inputSchema: {
          type: "object",
          properties: {
            operation: {
              type: "string",
              enum: ["add", "subtract", "multiply", "divide"],
              description: "実行する演算",
            },
            a: {
              type: "number",
              description: "最初の数値",
            },
            b: {
              type: "number",
              description: "二番目の数値",
            },
          },
          required: ["operation", "a", "b"],
        },
      },
    ],
  };
});

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  switch (name) {
    case "greet": {
      const { name: userName } = args as { name: string };
      return {
        content: [
          {
            type: "text",
            text: `こんにちは、${userName}さん！`,
          },
        ],
      };
    }
    case "calculate": {
      const { operation, a, b } = args as {
        operation: string;
        a: number;
        b: number;
      };
      let result: number;
      switch (operation) {
        case "add":
          result = a + b;
          break;
        case "subtract":
          result = a - b;
          break;
        case "multiply":
          result = a * b;
          break;
        case "divide":
          if (b === 0) {
            throw new Error("0で割ることはできません");
          }
          result = a / b;
          break;
        default:
          throw new Error(`未知の演算: ${operation}`);
      }
      return {
        content: [
          {
            type: "text",
            text: `${a} ${operation} ${b} = ${result}`,
          },
        ],
      };
    }
    default:
      throw new Error(`未知のツール: ${name}`);
  }
});

const transport = new StdioServerTransport();
await server.connect(transport);
console.error("MCPサーバーが起動しました");
```

### 3.5 メインのエントリーポイント

`src/index.ts`を作成し、クライアントとサーバーを連携させます。

```typescript
import { MCPClient } from "./client.js";
import { fileURLToPath } from "url";
import { dirname, join } from "path";

async function main() {
  const client = new MCPClient();
  
  try {
    // サーバーのパスを指定（実際の環境に合わせて変更）
    const serverPath = join(
      dirname(fileURLToPath(import.meta.url)),
      "server.ts"
    );
    
    await client.connect(serverPath);
    
    // 利用可能なツールを一覧表示
    const tools = await client.listTools();
    console.log("利用可能なツール:");
    tools.forEach((tool) => {
      console.log(`  - ${tool.name}: ${tool.description}`);
    });
    
    // ツールを呼び出し
    const greetResult = await client.callTool("greet", { name: "太郎" });
    console.log("greet結果:", greetResult);
    
    const calcResult = await client.callTool("calculate", {
      operation: "multiply",
      a: 6,
      b: 7,
    });
    console.log("calculate結果:", calcResult);
    
  } catch (error) {
    console.error("エラーが発生しました:", error);
  } finally {
    await client.disconnect();
  }
}

main();
```

### 3.6 実行

```bash
bun run src/index.ts
```

実行結果の例:
```
MCPサーバーに接続しました
利用可能なツール:
  - greet: ユーザーに挨拶を返します
  - calculate: 簡単な計算を行います
greet結果: [ { type: 'text', text: 'こんにちは、太郎さん！' } ]
calculate結果: [ { type: 'text', text: '6 multiply 7 = 42' } ]
接続を切断しました
```

## 4. コードサンプル

### サンプル1: ファイル操作ツールを追加する

サーバーにファイル読み込みツールを追加します。

```typescript
// server.ts に追加
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      // 既存のツール定義...
      {
        name: "read_file",
        description: "指定されたファイルの内容を読み込みます",
        inputSchema: {
          type: "object",
          properties: {
            path: {
              type: "string",
              description: "読み込むファイルのパス",
            },
          },
          required: ["path"],
        },
      },
    ],
  };
});

// ツール呼び出しハンドラーに追加
case "read_file": {
  const { path } = args as { path: string };
  const content = await Bun.file(path).text();
  return {
    content: [
      {
        type: "text",
        text: content,
      },
    ],
  };
}
```

### サンプル2: エラーハンドリングを強化する

クライアント側でエラーハンドリングを追加します。

```typescript
// client.ts の callTool メソッドを拡張
async callTool(
  toolName: string,
  args: Record<string, unknown>
): Promise<unknown> {
  try {
    const result = await this.client.callTool({
      name: toolName,
      arguments: args,
    });
    
    if (result.isError) {
      console.error(`ツール ${toolName} がエラーを返しました`);
      return { error: result.content[0]?.text || "不明なエラー" };
    }
    
    return result.content;
  } catch (error) {
    console.error(`ツール ${toolName} の呼び出しに失敗:`, error);
    throw error;
  }
}
```

## 5. ハマりやすいポイント

### 5.1 トランスポートの設定

MCP SDKのバージョンによって、トランスポートの設定方法が異なる場合があります。`@modelcontextprotocol/sdk`のバージョン0.6以上では、`StdioClientTransport`のコンストラクタに直接ストリームを渡せます。

### 5.2 BunとNode.jsの互換性

BunはNode.jsのAPIを大部分サポートしていますが、一部のモジュールで互換性の問題が発生することがあります。特に`child_process`モジュールの代わりにBunの`spawn`関数を使用することを推奨します。

### 5.3 モジュール解決

BunではESM（ES Modules）がデフォルトです。`import`文でファイルを指定する際、拡張子（`.js`）を明示的に指定する必要があります。

### 5.4 エラーハンドリング

MCPサーバーがクラッシュした場合、クライアントは適切に再接続する必要があります。実運用では、再接続ロジックやタイムアウト処理を実装することを検討してください。

## 6. まとめ

この記事では、TypeScriptとBunを使用してMCPクライアントを実装する方法を解説しました。MCPを利用することで、AIモデルと外部ツールを安全かつ効率的に連携させることが可能になります。

実装のポイントは以下の通りです。

- MCP SDKを利用してクライアントとサーバーを構築する
- トランスポートにはStdioを使用し、プロセス間通信を実現する
- ツールの定義と呼び出しはスキーマベースで行う
- Bunの高速なランタイムを活かして効率的に動作させる

今回の実装をベースに、データベースアクセスや外部API連携など、より高度なツールを追加することで、実用的なAIエージェントの基盤を構築できます。

---

**あわせて読みたい**

AIエージェント開発のさらに深い実践知については、こちらのnote記事で詳しく解説しています。実務での自動化やチーム開発のヒントになれば幸いです。

[AIトレードbotが1,304円の損失から学んだこと：センチメントフィルターの重要性](https://note.com/agent_workshop/n/n369ed65b1a4d)