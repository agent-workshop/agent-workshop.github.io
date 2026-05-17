---
layout: post
title: "Claude Haikuで作るリアルタイムチャットボット（Node.js）"
date: 2026-05-17 09:00:00 +0900
categories: [Claude, WebSocket]
tags: [Claude, WebSocket, Node.js, TypeScript, チャットボット]
description: Claude Haikuで作るリアルタイムチャットボット（Node.js）の実装方法を解説します。
---
# Claude Haikuで作るリアルタイムチャットボット（Node.js）

## 1. 導入

近年、LLM（大規模言語モデル）を活用したチャットボットの需要が高まっています。特に、ユーザーからの入力に対して即座に応答を返す「リアルタイム性」は、カスタマーサポートや社内ヘルプデスクなどの実用的なシナリオで重要です。

本記事では、Anthropic社のClaude Haiku（最も低レイテンシなモデル）とNode.jsを使用して、サーバーサイドで動作するリアルタイムチャットボットの実装方法を解説します。Claude Haikuは、処理速度とコスト効率のバランスに優れており、リアルタイム応答が必要なユースケースに適しています。

この記事で構築するシステムは以下の特徴を持ちます：
- WebSocketを使用した双方向通信によるリアルタイム応答
- 会話履歴を保持するメモリ機能
- 環境変数による安全なAPIキー管理

## 2. 前提条件

### 必要な環境
- Node.js v18以上
- npm または yarn
- Anthropic APIキー（[console.anthropic.com](https://console.anthropic.com)から取得可能）

### 使用するライブラリ
- `@anthropic-ai/sdk` - Anthropic APIの公式SDK
- `ws` - WebSocketサーバーライブラリ
- `dotenv` - 環境変数の読み込み

## 3. 実装手順

### ステップ1: プロジェクトのセットアップ

```bash
mkdir claude-chatbot
cd claude-chatbot
npm init -y
npm install @anthropic-ai/sdk ws dotenv
```

### ステップ2: 環境変数の設定

プロジェクトルートに`.env`ファイルを作成します。

```env
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxx
PORT=8080
```

`.gitignore`に`.env`を追加するのを忘れないでください。

### ステップ3: サーバーコードの作成

`server.js`を作成し、WebSocketサーバーとClaude APIの統合を実装します。

## 4. コードサンプル

### サンプル1: 基本構造のサーバー

```javascript
// server.js
import { WebSocketServer } from 'ws';
import Anthropic from '@anthropic-ai/sdk';
import dotenv from 'dotenv';

dotenv.config();

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

const wss = new WebSocketServer({ port: process.env.PORT || 8080 });

console.log(`WebSocket server running on port ${process.env.PORT || 8080}`);

wss.on('connection', (ws) => {
  console.log('Client connected');
  
  // 会話履歴を保持する配列
  const conversationHistory = [];

  ws.on('message', async (data) => {
    try {
      const userMessage = data.toString();
      
      // ユーザーメッセージを履歴に追加
      conversationHistory.push({ role: 'user', content: userMessage });
      
      // Claude API を呼び出し
      const response = await anthropic.messages.create({
        model: 'claude-3-haiku-20240307',
        max_tokens: 1024,
        messages: conversationHistory,
      });

      const assistantMessage = response.content[0].text;
      
      // アシスタントの応答を履歴に追加
      conversationHistory.push({ role: 'assistant', content: assistantMessage });
      
      // クライアントに応答を送信
      ws.send(JSON.stringify({ type: 'response', content: assistantMessage }));
      
    } catch (error) {
      console.error('Error:', error);
      ws.send(JSON.stringify({ type: 'error', content: 'サーバーエラーが発生しました' }));
    }
  });

  ws.on('close', () => {
    console.log('Client disconnected');
  });
});
```

### サンプル2: クライアント側HTML（テスト用）

```html
<!-- client.html -->
<!DOCTYPE html>
<html>
<head>
  <title>Claude Chatbot</title>
  <style>
    body { font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; padding: 20px; }
    #messages { border: 1px solid #ccc; height: 400px; overflow-y: auto; padding: 10px; margin-bottom: 10px; }
    .user { color: blue; margin-bottom: 5px; }
    .assistant { color: green; margin-bottom: 5px; }
    #input { width: 80%; padding: 8px; }
    #send { padding: 8px 16px; }
  </style>
</head>
<body>
  <h1>Claude Haiku チャットボット</h1>
  <div id="messages"></div>
  <input type="text" id="input" placeholder="メッセージを入力..." />
  <button id="send">送信</button>

  <script>
    const ws = new WebSocket('ws://localhost:8080');
    const messages = document.getElementById('messages');
    const input = document.getElementById('input');
    const sendBtn = document.getElementById('send');

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      const div = document.createElement('div');
      
      if (data.type === 'response') {
        div.className = 'assistant';
        div.textContent = `アシスタント: ${data.content}`;
      } else if (data.type === 'error') {
        div.className = 'error';
        div.style.color = 'red';
        div.textContent = `エラー: ${data.content}`;
      }
      
      messages.appendChild(div);
      messages.scrollTop = messages.scrollHeight;
    };

    function sendMessage() {
      const message = input.value.trim();
      if (message) {
        const div = document.createElement('div');
        div.className = 'user';
        div.textContent = `あなた: ${message}`;
        messages.appendChild(div);
        
        ws.send(message);
        input.value = '';
      }
    }

    sendBtn.addEventListener('click', sendMessage);
    input.addEventListener('keypress', (e) => {
      if (e.key === 'Enter') sendMessage();
    });
  </script>
</body>
</html>
```

### サンプル3: 会話履歴の制限とエラーハンドリング

```javascript
// enhanced-server.js - 改善版サーバー
import { WebSocketServer } from 'ws';
import Anthropic from '@anthropic-ai/sdk';
import dotenv from 'dotenv';

dotenv.config();

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

const MAX_HISTORY = 20; // 最大会話ターン数

const wss = new WebSocketServer({ port: process.env.PORT || 8080 });

wss.on('connection', (ws) => {
  const conversationHistory = [];

  ws.on('message', async (data) => {
    try {
      const userMessage = data.toString();
      
      // メッセージが空の場合は無視
      if (!userMessage.trim()) {
        ws.send(JSON.stringify({ type: 'error', content: 'メッセージを入力してください' }));
        return;
      }

      // 入力値のバリデーション
      if (userMessage.length > 2000) {
        ws.send(JSON.stringify({ type: 'error', content: 'メッセージが長すぎます（最大2000文字）' }));
        return;
      }

      conversationHistory.push({ role: 'user', content: userMessage });

      // 履歴が制限を超えた場合、古いものから削除
      if (conversationHistory.length > MAX_HISTORY * 2) {
        const removeCount = conversationHistory.length - MAX_HISTORY * 2;
        conversationHistory.splice(0, removeCount);
      }

      const response = await anthropic.messages.create({
        model: 'claude-3-haiku-20240307',
        max_tokens: 1024,
        messages: conversationHistory,
      });

      const assistantMessage = response.content[0].text;
      conversationHistory.push({ role: 'assistant', content: assistantMessage });

      ws.send(JSON.stringify({ type: 'response', content: assistantMessage }));

    } catch (error) {
      console.error('API Error:', error);
      
      // エラータイプに応じたメッセージ
      if (error.status === 401) {
        ws.send(JSON.stringify({ type: 'error', content: 'APIキーが無効です' }));
      } else if (error.status === 429) {
        ws.send(JSON.stringify({ type: 'error', content: 'レート制限に達しました。しばらく待ってから再試行してください' }));
      } else {
        ws.send(JSON.stringify({ type: 'error', content: 'サーバーエラーが発生しました' }));
      }
    }
  });

  ws.on('close', () => {
    console.log('Client disconnected');
  });
});
```

## 5. ハマりやすいポイント

### 1. CORSとWebSocketの接続問題
ブラウザからWebSocket接続を行う場合、同一オリジンポリシーに注意が必要です。開発環境では問題ありませんが、本番環境では適切なCORS設定が必要です。

### 2. 会話履歴の管理
会話履歴を無制限に保持すると、APIコールのトークン数が増加し、コストと応答時間に影響します。適切な上限（サンプル3では20ターン）を設定しましょう。

### 3. APIキーの漏洩防止
`.env`ファイルは必ず`.gitignore`に追加し、公開リポジトリにコミットしないようにしてください。また、クライアントサイドでAPIキーを直接使用するのは避けましょう。

### 4. エラーハンドリングの重要性
API呼び出しは様々な理由で失敗する可能性があります（レート制限、認証エラー、ネットワーク障害など）。ユーザーに適切なエラーメッセージを返す処理を実装しましょう。

### 5. メモリリーク対策
長時間稼働するサーバーでは、切断されたクライアントの会話履歴がメモリに残り続ける可能性があります。クライアント切断時に履歴をクリアする処理を追加するとよいでしょう。

## 6. まとめ

本記事では、Claude HaikuとNode.jsを使用してリアルタイムチャットボットを構築する方法を解説しました。WebSocketによる双方向通信、会話履歴の管理、エラーハンドリングなど、実用的なチャットボットに必要な基本的な要素をカバーしました。

Claude Haikuは応答速度が速いため、リアルタイム性が求められるアプリケーションに適しています。今回の実装をベースに、以下のような拡張も可能です：
- データベースを使った永続的な会話履歴の保存
- 複数クライアント間での会話セッション管理
- ストリーミング応答の実装（よりリアルタイムなUX）

ぜひ実際にコードを動かして、動作を確認してみてください。

---

**あわせて読みたい**

AIエージェント開発のさらに深い実践知については、こちらのnote記事で詳しく解説しています。実務での自動化やチーム開発のヒントになれば幸いです。

[AIに1円稼いでと言ったら何が起きたか](https://note.com/agent_workshop/n/n74300b556a8d)