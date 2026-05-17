---
layout: post
title: "Anthropic APIのプロンプトキャッシュでコストを70%削減する"
date: 2026-05-17 09:00:00 +0900
categories: [Claude, Anthropic]
tags: [Claude, Anthropic, コスト最適化, API, 生成AI]
description: Anthropic APIのプロンプトキャッシュでコストを70%削減するの実装方法を解説します。
---
# Anthropic APIのプロンプトキャッシュでコストを70%削減する

大規模言語モデル（LLM）の活用が広がるにつれて、API利用コストは多くの開発者にとって重要な課題となっています。特に、同じようなプロンプトを繰り返し送信するケースでは、そのコストは無視できないものとなるでしょう。

本記事では、Anthropic APIが提供するプロンプトキャッシュ機能に着目し、これを活用することでAPIコストを大幅に削減する方法を解説します。具体的には、繰り返し発生する入力トークンの課金を削減し、場合によっては最大で70%以上のコスト削減効果を期待できます。

TypeScript/JavaScriptでの具体的な実装例を交えながら、プロンプトキャッシュの仕組み、活用方法、そして注意点までを深掘りしていきます。

## 1. 導入：なぜプロンプトキャッシュが必要なのか

LLMを用いたアプリケーションでは、ユーザーからの入力に対して、常に共通のシステムプロンプトや文脈情報（RAGで取得したドキュメントなど）を付与してAPIを呼び出すことがよくあります。このような場合、システムプロンプトや共通の文脈情報は、ユーザー入力が変わってもほとんど変化しません。

しかし、従来のAPI呼び出しでは、毎回同じシステムプロンプトや文脈情報を含めて送信するため、その都度、入力トークンとして課金されていました。これが、アプリケーションのスケールに伴い、APIコストが膨れ上がる主要な原因の一つとなります。

Anthropic APIのプロンプトキャッシュは、この課題を解決するための強力な機能です。プロンプト全体が過去に送信されたものと完全に一致する場合、Anthropicのシステムがその応答をキャッシュから返し、入力トークンの課金を大幅に削減します。これにより、特にシステムプロンプトの比重が高いリクエストや、RAGのように共通のドキュメントを何度も参照するようなアプリケーションで、顕著なコスト削減効果とレイテンシの改善が期待できます。

本記事では、このプロンプトキャッシュの仕組みを理解し、TypeScript/JavaScriptでどのようにその恩恵を受けるか、具体的なコードを通じて解説します。

## 2. 前提条件

本記事のコードサンプルを実行するには、以下の環境とライブラリが必要です。

-   **Node.js**: v18以上を推奨します。
-   **TypeScript**: Node.js環境でTypeScriptを使用します。
-   **Anthropic API Key**: [Anthropicのウェブサイト](https://console.anthropic.com/settings/keys)でAPIキーを取得してください。
-   **`@anthropic-ai/sdk`**: Anthropic APIを操作するための公式JavaScript/TypeScript SDKです。

## 3. 実装手順

まずは、プロジェクトのセットアップから始めましょう。

### 3.1. プロジェクトの初期化

新しいディレクトリを作成し、Node.jsプロジェクトを初期化します。

```bash
mkdir anthropic-cache-demo
cd anthropic-cache-demo
npm init -y
```

### 3.2. 必要なパッケージのインストール

TypeScriptとAnthropic SDKをインストールします。

```bash
npm install typescript @anthropic-ai/sdk
npm install -D @types/node
```

### 3.3. TypeScriptの設定

`tsconfig.json` ファイルを作成し、TypeScriptのコンパイル設定を行います。

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true,
    "outDir": "./dist"
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### 3.4. APIキーの設定

Anthropic APIキーは環境変数として設定することを強く推奨します。これにより、コードに直接キーを書き込むことによるセキュリティリスクを回避できます。

`.env` ファイルを作成し、APIキーを記述します。

```
ANTHROPIC_API_KEY=sk-your-anthropic-api-key-here
```

この環境変数を読み込むために、`dotenv` パッケージをインストールすることもできます。

```bash
npm install dotenv
```

そして、コードの冒頭で `dotenv.config()` を呼び出します。

### 3.5. Anthropic APIの基本呼び出し

Anthropic APIのプロンプトキャッシュは、開発者が特別な設定をしなくても、自動的に機能します。Anthropicのシステムが、過去に送信されたリクエストと完全に一致するプロンプトを検知した場合に、自動的にキャッシュされた応答を返します。

したがって、我々が実装すべきは、通常のAPI呼び出しであり、そのレスポンスからキャッシュヒットの状況を確認することです。

## 4. コードサンプル

ここからは、具体的なコードサンプルを通じて、プロンプトキャッシュの挙動を確認し、コスト削減効果をシミュレーションします。

### 4.1. サンプル1: 基本的なAPI呼び出しとキャッシュの挙動確認

このサンプルでは、同じプロンプトを複数回送信し、Anthropicのプロンプトキャッシュがどのように機能するかを確認します。

`src/cacheDemo.ts` ファイルを作成し、以下のコードを記述します。

```typescript
// src/cacheDemo.ts
import Anthropic from '@anthropic