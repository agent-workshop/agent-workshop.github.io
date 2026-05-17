---
layout: post
title: "OpenAI APIとClaude APIを組み合わせるマルチLLMアーキテクチャ"
date: 2026-05-17 09:00:00 +0900
categories: [Claude, OpenAI]
tags: [Claude, OpenAI, マルチLLM, TypeScript, AIエージェント]
description: OpenAI APIとClaude APIを組み合わせるマルチLLMアーキテクチャの実装方法を解説します。
---
# OpenAI APIとClaude APIを組み合わせるマルチLLMアーキテクチャ

## 1. 導入

LLM（大規模言語モデル）のAPIは、OpenAI（GPT-4o / GPT-4）やAnthropic（Claude 3.5 Sonnet）など複数のプロバイダーから提供されています。それぞれに得意分野があり、単一のモデルに依存するより、用途に応じて使い分けることで精度やコストを最適化できます。

たとえば、OpenAIのGPT-4はコード生成や構造化データの処理に強く、Claude 3.5 Sonnetは長文の要約や論理的な推論に優れています。また、OpenAIのEmbeddings APIとClaudeのテキスト生成を組み合わせることで、RAG（検索拡張生成）パイプラインをより柔軟に構築できます。

本記事では、TypeScript（Node.js）でOpenAI APIとClaude APIを同一アプリケーション内で統合し、タスクごとに最適なモデルを選択する「マルチLLMアーキテクチャ」の実装方法を解説します。

## 2. 前提条件

- Node.js 18以上（fetch APIが標準対応）
- TypeScript 5.0以上
- OpenAI APIキー（`OPENAI_API_KEY`）
- Anthropic APIキー（`ANTHROPIC_API_KEY`）
- 各ライブラリのインストール

```bash
npm install openai @anthropic-ai/sdk dotenv
npm install -D typescript @types/node ts-node
```

## 3. 実装手順

### ステップ1: 環境変数とクライアントの初期化

`.env`ファイルにAPIキーを設定し、クライアントを生成します。

### ステップ2: モデル選択ロジックの実装

タスクの種類（要約、コード生成、質問応答など）に応じて使用するモデルを切り替える関数を作成します。

### ステップ3: 実際のAPI呼び出し

OpenAIとClaudeのAPIをそれぞれ呼び出し、結果を統一した型で返すようにします。

### ステップ4: エラーハンドリングとフォールバック

片方のAPIがエラーを返した場合、もう片方にフォールバックする仕組みを追加します。

## 4. コードサンプル

### サンプル1: マルチLLMクライアントの基本実装

```typescript
import { OpenAI } from 'openai';
import Anthropic from '@anthropic-ai/sdk';
import dotenv from 'dotenv';

dotenv.config();

type TaskType = 'summary' | 'code' | 'qa' | 'embedding';

interface LLMResponse {
  content: string;
  model: string;
  provider: 'openai' | 'anthropic';
}

class MultiLLMClient {
  private openai: OpenAI;
  private anthropic: Anthropic;

  constructor() {
    this.openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
    this.anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
  }

  async generate(prompt: string, task: TaskType): Promise<LLMResponse> {
    if (task === 'summary') {
      return this.callClaude(prompt);
    } else if (task === 'code') {
      return this.callGPT4(prompt);
    } else {
      // デフォルトはOpenAI
      return this.callGPT4(prompt);
    }
  }

  private async callGPT4(prompt: string): Promise<LLMResponse> {
    const response = await this.openai.chat.completions.create({
      model: 'gpt-4o',
      messages: [{ role: 'user', content: prompt }],
      max_tokens: 1024,
    });
    return {
      content: response.choices[0]?.message?.content ?? '',
      model: 'gpt-4o',
      provider: 'openai',
    };
  }

  private async callClaude(prompt: string): Promise<LLMResponse> {
    const response = await this.anthropic.messages.create({
      model: 'claude-3-5-sonnet-20241022',
      max_tokens: 1024,
      messages: [{ role: 'user', content: prompt }],
    });
    const content = response.content
      .filter((block) => block.type === 'text')
      .map((block) => (block as { text: string }).text)
      .join('');
    return {
      content,
      model: 'claude-3-5-sonnet',
      provider: 'anthropic',
    };
  }
}

// 使用例
async function main() {
  const client = new MultiLLMClient();

  const summaryResult = await client.generate(
    '次の文章を3行で要約してください：...',
    'summary'
  );
  console.log(`[${summaryResult.provider}] ${summaryResult.content}`);

  const codeResult = await client.generate(
    'TypeScriptで配列をシャッフルする関数を書いてください',
    'code'
  );
  console.log(`[${codeResult.provider}] ${codeResult.content}`);
}

main().catch(console.error);
```

### サンプル2: フォールバック付き呼び出し

```typescript
async function generateWithFallback(
  prompt: string,
  primaryProvider: 'openai' | 'anthropic'
): Promise<LLMResponse> {
  const client = new MultiLLMClient();

  try {
    if (primaryProvider === 'openai') {
      return await client.callGPT4(prompt);
    } else {
      return await client.callClaude(prompt);
    }
  } catch (error) {
    console.warn(`${primaryProvider} failed, falling back to the other.`);
    if (primaryProvider === 'openai') {
      return await client.callClaude(prompt);
    } else {
      return await client.callGPT4(prompt);
    }
  }
}
```

### サンプル3: 埋め込みと生成の連携（RAGパイプライン）

```typescript
async function ragQuery(query: string, documents: string[]): Promise<string> {
  const client = new MultiLLMClient();

  // OpenAIで埋め込み生成
  const embeddingResponse = await client['openai'].embeddings.create({
    model: 'text-embedding-3-small',
    input: [query, ...documents],
  });
  const queryEmbedding = embeddingResponse.data[0].embedding;
  const docEmbeddings = embeddingResponse.data.slice(1).map((d) => d.embedding);

  // コサイン類似度で最も近い文書を選択
  const similarities = docEmbeddings.map((docEmb) => {
    const dotProduct = queryEmbedding.reduce((sum, q, i) => sum + q * docEmb[i], 0);
    const normQ = Math.sqrt(queryEmbedding.reduce((s, v) => s + v * v, 0));
    const normD = Math.sqrt(docEmb.reduce((s, v) => s + v * v, 0));
    return dotProduct / (normQ * normD);
  });
  const bestDocIndex = similarities.indexOf(Math.max(...similarities));
  const context = documents[bestDocIndex];

  // Claudeで回答生成
  const prompt = `以下の情報に基づいて質問に答えてください。\n情報: ${context}\n質問: ${query}`;
  const result = await client.generate(prompt, 'qa');
  return result.content;
}
```

## 5. ハマりやすいポイント

### APIキーの管理
- `.env`ファイルをGit管理に含めないように注意。`.gitignore`に追加すること。
- 環境変数が未設定の場合、実行時にエラーになる。起動時にチェックするバリデーションを入れると安全。

### モデル名の指定
- Claude APIのモデル名は`claude-3-5-sonnet-20241022`のように日付付き。最新の安定版を確認する。
- OpenAIの`gpt-4o`は頻繁にアップデートされるが、互換性が保たれることが多い。

### レート制限
- 無料枠や低ティアのAPIキーでは、1分あたりのリクエスト数に制限がある。直列実行より並列実行の場合は`p-limit`ライブラリで制御する。

### トークン数の管理
- 応答が長くなると`max_tokens`を超えて途中で切れる場合がある。プロンプトに「最大500文字以内で」などの指示を入れると安定する。

## 6. まとめ

OpenAI APIとClaude APIを組み合わせることで、各モデルの強みを活かした柔軟なLLMアプリケーションを構築できます。本記事で紹介したアーキテクチャは、タスクの種類に応じてモデルを切り替えたり、障害時にフォールバックしたりする基本的なパターンです。

実際のプロダクションでは、コスト管理（トークン使用量のログ）、レイテンシ計測、A/Bテストによるモデル評価などを追加すると、より実用的になります。また、両方のAPIを呼び出して結果を比較する「アンサンブル」パターンも検討価値があります。

---

**あわせて読みたい**

AIエージェント開発のさらに深い実践知については、こちらのnote記事で詳しく解説しています。実務での自動化やチーム開発のヒントになれば幸いです。

[5日間で収益率+7.4%を記録。AIエージェントが自律運用する「モメンタム×センチメント」ハイブリッド戦略の設計図](https://note.com/agent_workshop/n/n9ffc6b4b1b35)