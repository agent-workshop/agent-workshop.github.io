---
layout: post
title: "Claude Code でコードレビューを自動化する実践ガイド"
date: 2026-05-17 09:00:00 +0900
categories: [ClaudeCode, コードレビュー]
tags: [ClaudeCode, コードレビュー, 自動化, TypeScript, 開発効率]
description: Claude Code でコードレビューを自動化する実践ガイドの実装方法を解説します。
---
# Claude Code でコードレビューを自動化する実践ガイド

## 導入

コードレビューは品質維持に欠かせないプロセスですが、プルリクエストが増えるにつれて「細かいスタイルの指摘に時間を取られる」「レビュワーの負荷が偏る」といった課題が生じます。こうした状況で、Claude Codeを活用してレビュー作業の一部を自動化する方法を紹介します。

Claude Codeは、Anthropicが提供するClaudeモデルをベースにしたコード支援ツールです。コードレビューにおいては、型の不整合、未使用変数、パターンの違反、ドキュメント不足などを自動検出できます。完全に人間のレビューを代替するものではありませんが、ルーティンワークを削減し、レビュワーが本質的なロジックや設計に集中できる環境を作ります。

本記事では、TypeScriptプロジェクトを例に、Claude Codeを使ったレビュー自動化の具体的な手順を解説します。

## 前提条件

以下の環境を想定しています。

- Node.js 18以上
- npm または yarn
- TypeScript 5.0以上
- Claude Code CLI（インストール方法は後述）
- Git管理下のプロジェクト

APIキーの取得にはAnthropicのアカウント登録が必要ですが、本記事ではCLIツールのセットアップ手順も含めます。

## 実装手順

### 1. Claude Code CLIのインストールと設定

まず、CLIツールをグローバルインストールします。

```bash
npm install -g @anthropic-ai/claude-code
```

次に、プロジェクトルートで初期化を行います。

```bash
claude-code init
```

このコマンドで、`.claude-code/` ディレクトリと設定ファイルが生成されます。APIキーは環境変数 `ANTHROPIC_API_KEY` に設定してください。

```bash
export ANTHROPIC_API_KEY=sk-xxxxxxxxxxxxxxxx
```

### 2. レビュールールの定義

プロジェクトのルートに `.claude-code/rules/` ディレクトリを作成し、レビューのルールをYAMLで記述します。以下は基本的なルール例です。

```yaml
# .claude-code/rules/review-rules.yaml
review:
  rules:
    - id: "no-any-type"
      severity: "warning"
      description: "any型の使用を避け、具体的な型を定義すること"
      pattern: ": any"
    - id: "missing-return-type"
      severity: "error"
      description: "関数には戻り値の型を明示すること"
      check: "function.*{"
    - id: "no-console-log"
      severity: "info"
      description: "console.logはデバッグ用。本番コードではloggerを使用すること"
      pattern: "console\\.log"
```

### 3. レビュー実行用スクリプトの作成

プルリクエストの変更差分をレビューするスクリプトを作成します。

```typescript
// scripts/review.ts
import { execSync } from 'child_process';
import * as fs from 'fs';
import * as path from 'path';

async function main() {
  // 変更ファイルの一覧を取得
  const diffFiles = execSync('git diff --name-only HEAD~1')
    .toString()
    .trim()
    .split('\n')
    .filter(file => file.endsWith('.ts') || file.endsWith('.tsx'));

  if (diffFiles.length === 0) {
    console.log('レビュー対象のTypeScriptファイルはありません。');
    return;
  }

  console.log(`レビュー対象ファイル: ${diffFiles.length}件`);

  // ファイルごとにClaude Codeでレビューを実行
  for (const file of diffFiles) {
    const filePath = path.resolve(file);
    if (!fs.existsSync(filePath)) continue;

    console.log(`\n--- レビュー中: ${file} ---`);
    
    try {
      const result = execSync(
        `claude-code review --file "${filePath}" --format json`,
        { encoding: 'utf-8', timeout: 30000 }
      );
      const review = JSON.parse(result);
      console.log(review.summary);
      
      // 指摘事項があれば表示
      if (review.issues && review.issues.length > 0) {
        review.issues.forEach((issue: any) => {
          console.log(`  [${issue.severity}] ${issue.line}:${issue.column} - ${issue.message}`);
        });
      }
    } catch (error) {
      console.error(`レビュー失敗: ${file}`, error);
    }
  }
}

main().catch(console.error);
```

### 4. CIとの連携

GitHub Actionsで自動レビューを実行する設定例です。

```yaml
# .github/workflows/code-review.yml
name: Claude Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm install -g @anthropic-ai/claude-code
      - run: claude-code init
      - run: npx ts-node scripts/review.ts
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

## コードサンプル

### サンプル1: レビュー対象のコード（改善前）

```typescript
// src/services/userService.ts
function getUser(id: any) {
  const data = fetch(`/api/users/${id}`);
  console.log('User data:', data);
  return data;
}

function updateUser(id, name) {
  return fetch(`/api/users/${id}`, {
    method: 'PUT',
    body: JSON.stringify({ name })
  });
}
```

### サンプル2: レビュー後の修正コード

```typescript
// src/services/userService.ts
import { logger } from '../utils/logger';

interface User {
  id: string;
  name: string;
  email: string;
}

async function getUser(id: string): Promise<User | null> {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) {
    logger.error(`Failed to fetch user: ${id}`);
    return null;
  }
  const data: User = await response.json();
  return data;
}

async function updateUser(id: string, name: string): Promise<boolean> {
  const response = await fetch(`/api/users/${id}`, {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ name })
  });
  return response.ok;
}
```

### サンプル3: カスタムレビュールールの拡張例

```yaml
# .claude-code/rules/review-rules.yaml に追加
review:
  rules:
    - id: "async-function-no-await"
      severity: "error"
      description: "async関数内でawaitが使用されていない場合は不要なasyncを削除すること"
      pattern: "async function.*\\{\\s*\\n(?!.*await)"
    - id: "magic-number"
      severity: "warning"
      description: "マジックナンバーは定数として定義すること"
      pattern: "(?<![a-zA-Z])\\b\\d{2,}\\b(?![a-zA-Z])"
    - id: "missing-error-handling"
      severity: "error"
      description: "非同期処理にはtry-catchまたは.catch()によるエラーハンドリングが必要"
      check: "await\\s+.*fetch|await\\s+.*axios"
```

## ハマりやすいポイント

### 1. 差分の検出範囲
`git diff HEAD~1` は直前のコミットとの差分を見ます。プルリクエスト全体を対象にする場合は、ベースブランチとの差分を指定する必要があります。GitHub Actionsでは `github.event.pull_request.base.sha` を利用するとよいでしょう。

### 2. ファイルが削除された場合のエラー
`git diff` で取得したファイルが既に削除されているケースがあります。`fs.existsSync` で存在確認を行うことで回避できます。

### 3. タイムアウト設定
大規模なファイルや複雑なコードではClaude Codeの応答に時間がかかることがあります。`execSync` の `timeout` オプションは適切な値（30秒〜60秒）に設定しましょう。

### 4. APIキーの管理
CI環境ではSecretsにAPIキーを保存しますが、ローカルで試す際は `.env` ファイルの利用や `.gitignore` への追加を忘れずに行ってください。

### 5. JSONパースの失敗
`--format json` オプションを付けないと、標準出力がプレーンテキストになります。CIのログ解析にはJSON形式が適しています。

## まとめ

Claude Codeを使ったコードレビューの自動化は、以下のメリットがあります。

- 型定義の不備や未使用変数などの機械的なチェックを自動化できる
- カスタムルールでプロジェクト固有の規約を強制できる
- CIに組み込むことで、プルリクエスト作成時に即座にフィードバックが得られる
- 人間のレビュワーは設計やロジックのレビューに集中できる

ただし、完全な自動化は現実的ではありません。ビジネスロジックの正確性やアーキテクチャの適切性は、人間の判断が必要です。Claude Codeはあくまで「一次レビュー」として位置づけ、最終的な判断はチームのレビュワーが行う運用をおすすめします。

導入は比較的簡単で、ルールファイルの整備とCI設定だけで始められます。まずは小さなプロジェクトで試し、徐々にルールを拡充していくとよいでしょう。

---

**あわせて読みたい**

AIエージェント開発のさらに深い実践知については、こちらのnote記事で詳しく解説しています。実務での自動化やチーム開発のヒントになれば幸いです。

[5日間で収益率+7.4%を記録。AIエージェントが自律運用する「モメンタム×センチメント」ハイブリッド戦略の設計図](https://note.com/agent_workshop/n/n9ffc6b4b1b35)