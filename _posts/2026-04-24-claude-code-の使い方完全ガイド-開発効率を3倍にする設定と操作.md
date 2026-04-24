---
layout: post
title: "Claude Code の使い方完全ガイド：開発効率を3倍にする設定と操作"
date: 2026-04-24 09:00:00 +0900
categories: [ClaudeCode, Claude]
tags: [ClaudeCode, Claude, 開発効率, AI, TypeScript]
description: Claude Code の使い方完全ガイド：開発効率を3倍にする設定と操作の実装方法を解説します。
---
# Claude Code の使い方完全ガイド：開発効率を3倍にする設定と操作

Claude Codeは、ターミナルの中でコードベースを読み、編集し、コマンド実行まで進められる開発向けエージェントです。チャットUIよりも「今あるリポジトリの文脈を使って作業する」ことに寄っているので、調査、修正、レビューの往復がかなり短くなります。

この記事では、中級エンジニア向けに、Claude Codeを使い始めるときに押さえておきたい設定と操作をまとめます。対象はTypeScriptやJavaScriptに慣れている人です。MCPやカスタムコマンドまで含めて、実務で効く範囲に絞ります。

## まず最初にやること

公式ドキュメントでは、インストールはnpm経由、要件はNode.js 18以上です。導入後は `claude doctor` で状態確認できます。

```bash
npm install -g @anthropic-ai/claude-code
claude doctor
```

プロジェクトで使い始めるなら、最初にリポジトリ直下で `claude` を起動してから `/init` を試すのが早いです。これで `CLAUDE.md` をベースに、そのリポジトリ向けの運用ルールを持たせやすくなります。

## 設定ファイルの置き場所を先に理解する

Claude Codeの設定は1か所ではありません。ここを最初に整理しておくと混乱しにくいです。

- `~/.claude/settings.json`: 全プロジェクト共通の設定
- `.claude/settings.json`: リポジトリ共有の設定
- `.claude/settings.local.json`: 個人用のローカル設定
- `CLAUDE.md`: 作業ルールや文脈を渡すメモリ

設定値そのものは `/config` でも編集できますが、実務では「共有したい設定は `.claude/settings.json`、個人差があるものは `.claude/settings.local.json`」と分けるだけでだいぶ運用しやすくなります。

## よく使う操作は slash command で覚える

Claude Codeは通常の会話だけでも動きますが、よく使う操作は slash command を覚えたほうが速いです。特に使う頻度が高いのはこのあたりです。

- `/init`: プロジェクト初期化
- `/config`: 設定確認と変更
- `/mcp`: MCP接続の確認
- `/permissions`: 権限設定の確認
- `/review`: コードレビュー依頼
- `/status`: アカウントや状態確認

たとえばMCPを使うなら、接続後に `/mcp` でサーバーの状態を見る、という流れが分かりやすいです。設定がずれているのに会話だけで原因を追うより早いです。

## カスタムコマンドを作ると反復作業が減る

同じ依頼を何度も打つなら、`.claude/commands/` にMarkdownを置いてカスタムコマンド化したほうが楽です。たとえばTypeScriptのPRレビュー用コマンドならこう書けます。

```md
---
description: Review TypeScript changes
---

TypeScriptの変更をレビューして、バグ、設計のズレ、テスト不足を優先順に指摘してください。
```

この内容を `.claude/commands/ts-review.md` に置くと、`/ts-review` で呼べます。チームで共有したい運用は、この形に寄せるとブレが減ります。

## 設定より効くのは、依頼の切り方

Claude Codeを使っていて詰まりやすいのは、設定不足より依頼が広すぎるケースです。最初から「全部直して」と渡すより、調査、方針確認、修正、テストの4段階に分けたほうが安定します。

たとえばTypeScriptの不具合修正なら、最初の依頼はこれくらいで十分です。

```text
このエラーの原因候補を3つに絞って。関連ファイルも挙げて。
```

そこから原因が見えた段階で、「この関数だけ直して」「影響範囲のテストも追加して」と刻むと、無駄な変更が減ります。Claude Codeは大きな曖昧依頼もこなせますが、実務では小さく区切ったほうがレビューしやすいです。

## MCPを足すと調査がかなり速くなる

Claude Codeは `/mcp` でMCPサーバーを扱えます。Issue検索、社内API、デプロイ状態確認のような外部情報をMCPでつなぐと、別タブへ移動する回数が減ります。

最初は読み取り専用ツールから始めるのが安全です。いきなり書き込み系までつなぐより、まず検索や参照だけを道具にしたほうが事故が少ないです。

## 実務でのおすすめ運用

最後に、使い始めで効きやすい運用を3つだけ挙げます。

1. リポジトリごとに `CLAUDE.md` を置く
2. 繰り返す依頼は slash command 化する
3. 個人設定と共有設定を分ける

Claude Codeは「高機能なチャット」として使うより、「リポジトリに住みつく作業用インターフェース」として使ったほうが強みが出ます。設定を増やす前に、よく使う操作を決めて固定化するのが近道です。

## まとめ

Claude Codeを使い始めるときは、インストール後に `claude doctor` で状態を確認し、`/init` でプロジェクト文脈を作り、`/config` と `/mcp` の位置づけを理解しておくと入りやすいです。

そのうえで、設定ファイルの置き場、slash command、MCPの3つを押さえると、日々の反復作業をかなり減らせます。最初から全部盛りにせず、レビューや調査のような頻出作業から寄せていくのが現実的です。

---

## 関連記事

- [MCPサーバーを30分で自作する（TypeScript）](https://agent-workshop.github.io/2026/04/14/mcp-server-typescript/)
- [Claude APIでRAGを実装する最小構成（TypeScript）](https://agent-workshop.github.io/2026/04/15/claude-apiでragを実装する最小構成-typescript-/)
- [MCPを使ってClaude Codeに外部APIを接続する](https://agent-workshop.github.io/2026/04/22/mcpを使ってclaude-codeに外部apiを接続する/)
