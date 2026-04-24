---
layout: post
title: "MCPを使ってClaude Codeに外部APIを接続する"
date: 2026-04-22 09:00:00 +0900
categories: [MCP, ClaudeCode]
tags: [MCP, ClaudeCode, API連携, TypeScript, AIエージェント]
description: MCPを使ってClaude Codeに外部APIを接続するの実装方法を解説します。
---
# MCPを使ってClaude Codeに外部APIを接続する

Claude Codeで開発していると、チケット管理、社内ダッシュボード、監視APIなどの情報を毎回手で貼り付けたくなる場面があります。

この手作業を減らす方法の1つがMCP（Model Context Protocol）です。MCPサーバーを用意すると、Claude Code側から決まったツールとして外部APIを呼び出せます。

この記事では、TypeScriptで小さなMCPサーバーを作り、外部APIの検索結果をClaude Codeから使えるようにする最小構成を紹介します。

## MCPで接続する構成

最小構成は次の3つです。

1. 外部APIを呼ぶTypeScript関数
2. その関数をMCPツールとして公開するサーバー
3. Claude CodeからそのMCPサーバーを起動する設定

ローカル開発では、まずstdio transportで十分です。Claude CodeがMCPサーバーを子プロセスとして起動し、標準入出力でJSON-RPCをやり取りします。HTTPサーバーを立てなくてよいので、認証やポート管理を後回しにできます。

## セットアップ

適当なディレクトリを作って依存関係を入れます。

```bash
mkdir claude-code-mcp-api
cd claude-code-mcp-api
npm init -y
npm install @modelcontextprotocol/sdk zod
npm install -D tsx typescript @types/node
```

`package.json` に起動コマンドを追加します。

```json
{
  "type": "module",
  "scripts": {
    "start": "tsx src/server.ts"
  }
}
```

## 外部APIを呼ぶ関数を作る

例として、外部のタスク管理APIを検索する想定にします。実際のAPIに合わせてURLやレスポンス型を差し替えてください。

```ts
// src/api.ts
export type TaskSummary = {
  id: string;
  title: string;
  status: string;
  updatedAt: string;
};

export async function searchTasks(query: string): Promise<TaskSummary[]> {
  const baseUrl = process.env.TASKS_API_BASE_URL;
  const token = process.env.TASKS_API_TOKEN;

  if (!baseUrl || !token) {
    throw new Error("TASKS_API_BASE_URL and TASKS_API_TOKEN are required");
  }

  const url = new URL("/tasks/search", baseUrl);
  url.searchParams.set("q", query);

  const res = await fetch(url, {
    headers: {
      Authorization: `Bearer ${token}`,
      Accept: "application/json",
    },
  });

  if (!res.ok) {
    throw new Error(`API request failed: ${res.status} ${res.statusText}`);
  }

  const data = await res.json() as { tasks: TaskSummary[] };
  return data.tasks.slice(0, 5);
}
```

ポイントは、APIキーをコードに書かないことです。Claude Codeに渡すMCP設定でも、値そのものではなく環境変数を参照する形にします。

## MCPサーバーとして公開する

次に、`searchTasks()` をMCPツールとして公開します。

```ts
// src/server.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import { searchTasks } from "./api.js";

const server = new McpServer({
  name: "tasks-api",
  version: "1.0.0",
});

server.registerTool(
  "search_tasks",
  {
    title: "Search tasks",
    description: "Search external task records by keyword",
    inputSchema: {
      query: z.string().min(1).describe("Search keyword"),
    },
  },
  async ({ query }) => {
    const tasks = await searchTasks(query);

    return {
      content: [
        {
          type: "text",
          text: JSON.stringify(tasks, null, 2),
        },
      ],
    };
  },
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

MCPツールの戻り値は、Claude Codeが読めるテキストとして返します。最初はJSON文字列で十分です。必要になったら、Markdownの表に整形したり、IDと要約だけ返したりすると使いやすくなります。

## Claude Codeに接続する

プロジェクト単位で共有したい場合は、リポジトリ直下に `.mcp.json` を置きます。

```json
{
  "mcpServers": {
    "tasks-api": {
      "type": "stdio",
      "command": "npm",
      "args": ["run", "start"],
      "env": {
        "TASKS_API_BASE_URL": "${TASKS_API_BASE_URL}",
        "TASKS_API_TOKEN": "${TASKS_API_TOKEN}"
      }
    }
  }
}
```

これでClaude Code側からMCPサーバーを読み込めます。接続状態はClaude Code内の `/mcp` で確認できます。

個人だけで使う実験用なら、CLIでローカル設定に追加する方法もあります。

```bash
claude mcp add-json tasks-api \
  '{"type":"stdio","command":"npm","args":["run","start"],"env":{"TASKS_API_BASE_URL":"${TASKS_API_BASE_URL}","TASKS_API_TOKEN":"${TASKS_API_TOKEN}"}}'
```

## 使い方の例

接続できたら、Claude Codeに次のように依頼できます。

```text
search_tasksで「billing webhook」を検索して、関連しそうな未完了タスクを要約して
```

Claude CodeはMCPツールを呼び、外部APIの結果を読んだうえで作業できます。チケット番号や仕様メモを毎回貼り付けるより、情報の取り違えを減らしやすくなります。

## 運用時の注意点

MCPは便利ですが、外部APIをそのまま操作できるようにすると影響範囲が広がります。最初は読み取り専用のツールから始めるのが安全です。

特に次の点は先に決めておくと運用しやすいです。

- APIキーは環境変数で渡し、リポジトリに保存しない
- 書き込み系ツールは、人間の確認を挟む設計にする
- 返すデータは必要最小限にする
- エラー時にトークンや個人情報をログへ出さない
- ツール名とdescriptionは、できるだけ具体的に書く

Claude Codeにとって、MCPツールの説明文は「いつ使うべきか」を判断する材料になります。`call_api` のような曖昧な名前より、`search_tasks` や `get_deploy_status` のように用途がわかる名前のほうが安定します。

## まとめ

Claude Codeに外部APIを接続するなら、まずはstdioのMCPサーバーを1つ作るのが手軽です。

TypeScript SDKで外部API呼び出しをツール化し、`.mcp.json` でClaude Codeに接続すれば、手で情報を貼り付ける作業を減らせます。最初は読み取り専用、小さいレスポンス、環境変数での認証から始めると、あとで書き込み操作やHTTP transportへ広げやすくなります。

---

## 関連記事

- [MCPサーバーを30分で自作する（TypeScript）](https://agent-workshop.github.io/2026/04/14/mcp-server-typescript/)
- [Claude APIでRAGを実装する最小構成（TypeScript）](https://agent-workshop.github.io/2026/04/15/claude-apiでragを実装する最小構成-typescript-/)
- [Claude Code の使い方完全ガイド：開発効率を3倍にする設定と操作](https://agent-workshop.github.io/2026/04/24/claude-code-の使い方完全ガイド-開発効率を3倍にする設定と操作/)
