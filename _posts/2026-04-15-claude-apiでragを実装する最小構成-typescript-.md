---
layout: post
title: "Claude APIでRAGを実装する最小構成（TypeScript）"
date: 2026-04-15 09:00:00 +0900
categories: [Claude, RAG]
tags: [Claude, RAG, TypeScript, 生成AI, LLM]
description: Claude APIでRAGを実装する最小構成（TypeScript）の実装方法を解説します。
---
# Claude APIでRAGを実装する最小構成（TypeScript）

## はじめに

RAG（Retrieval-Augmented Generation）を実装しようとすると、LangChainやLlamaIndexなどのフレームワークを使うのが一般的ですが、ライブラリの使い分けやバージョンの変化に戸惑うことも少なくありません。

実は、Claude API（Anthropic API）を直接使えば、驚くほどシンプルなコードで、かつ精度の高いRAGを実装できます。
この記事では、外部ライブラリを最小限に抑え、TypeScriptとBunを使って「Claude APIで動くRAGの最小構成」を構築する手順を解説します。

## 解決したい課題

- RAGの仕組みを基礎から理解したい
- 特定のフレームワークに依存せず、軽量なRAGを実装したい
- Claudeの長いコンテキスト（200k tokens）を最大限に活かしたい

## RAGの基本フロー

1. **ドキュメントの準備**: 検索対象となるテキストデータを準備します。
2. **チャンク分割**: テキストを適切な長さに分割します。
3. **埋め込み（Embedding）生成**: 各チャンクをベクトルデータに変換します。
4. **検索（Retrieval）**: ユーザーの質問に近いチャンクを検索します（今回はシンプルにキーワード・類似度で実装）。
5. **生成（Generation）**: 検索結果をコンテキストとしてClaudeに渡し、回答を得ます。

## 実装手順

### 1. プロジェクトの準備

```bash
mkdir claude-rag-demo
cd claude-rag-demo
bun init
bun add @anthropic-ai/sdk zod
```

### 2. ベクトル検索の簡易実装

今回はデータベースを使わず、メモリ上での簡易的なベクトル類似度計算（コサイン類似度）を行います。

```typescript
// src/utils/vector.ts
export function cosineSimilarity(a: number[], b: number[]): number {
  const dotProduct = a.reduce((sum, val, i) => sum + val * b[i], 0);
  const magnitudeA = Math.sqrt(a.reduce((sum, val) => sum + val * val, 0));
  const magnitudeB = Math.sqrt(b.reduce((sum, val) => sum + val * val, 0));
  return dotProduct / (magnitudeA * magnitudeB);
}
```

### 3. RAGのコアロジック

```typescript
import Anthropic from "@anthropic-ai/sdk";

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

interface Chunk {
  text: string;
  embedding: number[];
}

// ドキュメント群から最適なコンテキストを検索
async function retrieveContext(query: string, chunks: Chunk[]): Promise<string> {
  // 本来はqueryのembeddingを取得しますが、
  // ここではシンプルに「質問に含まれる単語」でマッチングする簡易検索例を示します。
  // (実際の実装では OpenAI や Voyage AI の Embedding API を推奨)
  return chunks
    .filter(chunk => chunk.text.includes(query.slice(0, 5))) // 簡易マッチング
    .map(chunk => chunk.text)
    .join("\n---\n");
}

async function askClaudeWithRAG(question: string, chunks: Chunk[]) {
  const context = await retrieveContext(question, chunks);

  const prompt = `以下の情報を参考にして、質問に答えてください。
情報のソースが不明な場合は、無理に答えず「わかりません」と伝えてください。

---
【参考情報】
${context}
---

質問: ${question}`;

  const message = await anthropic.messages.create({
    model: "claude-3-5-sonnet-latest",
    max_tokens: 1024,
    messages: [{ role: "user", content: prompt }],
  });

  return message.content;
}
```

## ハマりやすいポイント

- **チャンクサイズ**: 短すぎると文脈が切れ、長すぎると無関係な情報が混ざります。500〜1000文字程度が一般的です。
- **Claudeの特性**: Claudeは長いコンテキストに強いため、RAGで絞り込む際も、少し余裕を持って（周辺情報を含めて）コンテキストを渡すと回答の質が上がります。

## まとめ

RAGはフレームワークなしでも、基本的なフローを理解すれば数行のコードで実装可能です。
まずは最小構成で動かしてみて、そこからベクターストアの導入やランキングアルゴリズムの改善（Reranking）などにステップアップしていくのが、上達への近道です。

ぜひ、自作のドキュメントで試してみてください！

---

## 関連記事

- [MCPサーバーを30分で自作する（TypeScript）](https://agent-workshop.github.io/2026/04/14/mcp-server-typescript/)
- [MCPを使ってClaude Codeに外部APIを接続する](https://agent-workshop.github.io/2026/04/22/mcpを使ってclaude-codeに外部apiを接続する/)
- [Claude Code の使い方完全ガイド：開発効率を3倍にする設定と操作](https://agent-workshop.github.io/2026/04/24/claude-code-の使い方完全ガイド-開発効率を3倍にする設定と操作/)
