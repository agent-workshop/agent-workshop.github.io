---
layout: post
title: "MCPサーバーを30分で自作する（TypeScript + Bun）"
date: 2026-04-14 09:00:00 +0900
categories: [mcp, typescript, ai-agent]
tags: [MCP, TypeScript, Bun, Claude, AIエージェント]
description: Model Context Protocol（MCP）サーバーをTypeScript + Bunで自作する手順を解説します。Claude Codeに外部APIやデータソースを接続する最短ルートです。
---

## MCPとは何か

MCP（Model Context Protocol）は、LLMアプリケーションに外部ツールやデータソースを接続するためのオープンプロトコルです。Claude Codeに自作のツールを追加したり、外部APIを呼び出せるようにする仕組みです。

このガイドでは、TypeScript + BunでMCPサーバーを実装し、Claude Codeから利用できるようにします。

## 前提条件

- Bun 1.0以上
- Claude Codeインストール済み

## セットアップ

```bash
mkdir my-mcp-server && cd my-mcp-server
bun init -y
bun add @modelcontextprotocol/sdk
```

## 最小構成のMCPサーバー

```typescript
// server.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "my-mcp-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// ツール一覧を返す
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "get_current_time",
      description: "現在時刻を返す",
      inputSchema: {
        type: "object",
        properties: {},
      },
    },
  ],
}));

// ツール実行
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "get_current_time") {
    return {
      content: [
        {
          type: "text",
          text: new Date().toLocaleString("ja-JP", { timeZone: "Asia/Tokyo" }),
        },
      ],
    };
  }
  throw new Error(`Unknown tool: ${request.params.name}`);
});

// 起動
const transport = new StdioServerTransport();
await server.connect(transport);
```

## Claude Codeへの登録

`~/.claude.json` の `mcpServers` に追加します：

```json
{
  "mcpServers": {
    "my-mcp-server": {
      "command": "bun",
      "args": ["run", "/path/to/my-mcp-server/server.ts"]
    }
  }
}
```

Claude Codeを再起動すると `get_current_time` ツールが使えるようになります。

## 実用的なツールの追加例

外部APIを叩くツールに拡張する例：

```typescript
// ファイル読み書きツール
{
  name: "read_file",
  description: "指定パスのファイルを読む",
  inputSchema: {
    type: "object",
    properties: {
      path: { type: "string", description: "ファイルパス" },
    },
    required: ["path"],
  },
}

// ハンドラ
if (request.params.name === "read_file") {
  const { path } = request.params.arguments as { path: string };
  const content = await Bun.file(path).text();
  return { content: [{ type: "text", text: content }] };
}
```

## ハマりポイント

**1. stdioを汚染しない**
MCPはstdio経由で通信します。`console.log` をサーバーコード内で使うと通信が壊れます。デバッグには `console.error` を使いましょう。

```typescript
// NG
console.log("デバッグ"); // stdioを汚染する

// OK
console.error("デバッグ"); // stderrに出力
```

**2. 非同期処理のエラーハンドリング**
ツールハンドラが例外を投げると接続が切れることがあります。try-catchで囲んでエラーを返す形にしましょう。

```typescript
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  try {
    // ツール処理
  } catch (err) {
    return {
      content: [{ type: "text", text: `エラー: ${err}` }],
      isError: true,
    };
  }
});
```

**3. `~/.claude.json` への登録先を間違えない**
`~/.claude/settings.json` ではなく `~/.claude.json` の `mcpServers` が正しい登録先です。

## まとめ

MCPサーバーはBun + TypeScriptで30分あれば動かせます。Claude Codeに自分のユースケースに合ったツールを追加することで、AIの実用性が大きく広がります。

---

## 関連記事

- [Claude APIでRAGを実装する最小構成（TypeScript）](https://agent-workshop.github.io/2026/04/15/claude-apiでragを実装する最小構成-typescript-/)
- [MCPを使ってClaude Codeに外部APIを接続する](https://agent-workshop.github.io/2026/04/22/mcpを使ってclaude-codeに外部apiを接続する/)
- [Claude Code の使い方完全ガイド：開発効率を3倍にする設定と操作](https://agent-workshop.github.io/2026/04/24/claude-code-の使い方完全ガイド-開発効率を3倍にする設定と操作/)
