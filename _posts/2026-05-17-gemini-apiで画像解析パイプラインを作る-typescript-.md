---
layout: post
title: "Gemini APIで画像解析パイプラインを作る（TypeScript）"
date: 2026-05-17 09:00:00 +0900
categories: [Gemini, 画像解析]
tags: [Gemini, 画像解析, TypeScript, マルチモーダル, 生成AI]
description: Gemini APIで画像解析パイプラインを作る（TypeScript）の実装方法を解説します。
---
# Gemini APIで画像解析パイプラインを作る（TypeScript）

## 1. 導入

画像解析のユースケースは、プロダクト開発の現場で増えています。たとえば、ECサイトの商品画像からタグを自動生成する、アップロードされた画像の不適切な内容を検出する、OCRでテキストを抽出してデータベースに登録する――こうした処理を実装しようとすると、従来は機械学習モデルの選定、学習、デプロイまで含めた大きな工数が必要でした。

しかし、Gemini APIを利用すれば、数行のコードで画像解析を実現できます。特に、複数の画像を連続処理する「パイプライン」を組むことで、大量の画像データに対しても一貫した解析ロジックを適用できます。

本記事では、TypeScriptでGemini APIを使った画像解析パイプラインを構築する方法を、実際に動作するコードとともに解説します。

## 2. 前提条件

以下の環境・ライブラリを想定しています。

- Node.js 18以上（fetch APIが利用可能）
- TypeScript 5.0以上
- npmまたはyarn

必要なライブラリは、Google AI StudioのSDKである`@google/generative-ai`と、画像ファイルの読み込みに`fs`（Node.js標準）を使います。

```bash
npm install @google/generative-ai
```

また、Google AI StudioでAPIキーを取得し、環境変数`GEMINI_API_KEY`に設定しておいてください。

## 3. 実装手順

パイプラインの流れは以下の通りです。

1. 画像ファイルをBase64エンコードする
2. 解析用のプロンプトを構築する
3. Gemini APIにリクエストを送信する
4. レスポンスをパースして結果を取得する
5. 複数画像をループ処理する

ポイントは、画像データをインラインで送信できることです。ファイルパスを指定するのではなく、Base64文字列としてリクエストに含めます。

## 4. コードサンプル

### サンプル1: 単一画像の解析（基本形）

まずは、1枚の画像を解析して説明文を生成する基本形です。

```typescript
import { GoogleGenerativeAI } from "@google/generative-ai";
import * as fs from "fs";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);
const model = genAI.getGenerativeModel({ model: "gemini-1.5-pro" });

async function analyzeImage(imagePath: string): Promise<string> {
  // 画像をBase64に変換
  const imageBuffer = fs.readFileSync(imagePath);
  const base64Image = imageBuffer.toString("base64");

  const result = await model.generateContent([
    {
      inlineData: {
        mimeType: "image/jpeg",
        data: base64Image,
      },
    },
    "この画像に写っているものを日本語で3つ挙げてください。",
  ]);

  const response = await result.response;
  return response.text();
}

// 使用例
analyzeImage("./sample.jpg")
  .then((text) => console.log(text))
  .catch((err) => console.error(err));
```

このコードは、画像を読み込んでBase64に変換し、Gemini APIに「この画像に写っているものを3つ挙げて」というプロンプトとともに送信します。戻り値はマークダウン形式のテキストで、画像内のオブジェクトが列挙されます。

### サンプル2: 複数画像のバッチ処理（パイプライン）

次に、複数の画像を連続処理するパイプラインです。`Promise.allSettled`を使うことで、一部の画像でエラーが発生しても処理を継続できます。

```typescript
import { GoogleGenerativeAI } from "@google/generative-ai";
import * as fs from "fs";
import * as path from "path";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);
const model = genAI.getGenerativeModel({ model: "gemini-1.5-pro" });

interface AnalysisResult {
  fileName: string;
  success: boolean;
  text?: string;
  error?: string;
}

async function analyzeSingleImage(filePath: string): Promise<string> {
  const imageBuffer = fs.readFileSync(filePath);
  const base64Image = imageBuffer.toString("base64");

  const result = await model.generateContent([
    {
      inlineData: {
        mimeType: "image/jpeg",
        data: base64Image,
      },
    },
    "以下のJSON形式で出力してください。キーは「objects」「colors」「text」とします。",
  ]);

  const response = await result.response;
  return response.text();
}

async function batchAnalyzeImages(
  imageDir: string,
  extensions: string[] = [".jpg", ".jpeg", ".png"]
): Promise<AnalysisResult[]> {
  const files = fs.readdirSync(imageDir).filter((file) =>
    extensions.includes(path.extname(file).toLowerCase())
  );

  const promises = files.map(async (fileName) => {
    const filePath = path.join(imageDir, fileName);
    try {
      const text = await analyzeSingleImage(filePath);
      return { fileName, success: true, text };
    } catch (error) {
      return {
        fileName,
        success: false,
        error: error instanceof Error ? error.message : "Unknown error",
      };
    }
  });

  const results = await Promise.allSettled(promises);
  return results.map((r) =>
    r.status === "fulfilled" ? r.value : { fileName: "unknown", success: false, error: r.reason }
  );
}

// 使用例
batchAnalyzeImages("./images").then((results) => {
  results.forEach((r) => {
    if (r.success) {
      console.log(`✅ ${r.fileName}: ${r.text?.slice(0, 50)}...`);
    } else {
      console.error(`❌ ${r.fileName}: ${r.error}`);
    }
  });
});
```

このパイプラインは、指定されたディレクトリ内の画像ファイルをすべて解析し、結果を配列で返します。エラーが発生したファイルは`success: false`として記録されるため、後続の処理で再試行やスキップが可能です。

### サンプル3: カスタムプロンプトで解析内容を制御

解析内容を動的に変更したい場合、プロンプトを関数の引数として受け取れるようにします。

```typescript
import { GoogleGenerativeAI } from "@google/generative-ai";
import * as fs from "fs";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);
const model = genAI.getGenerativeModel({ model: "gemini-1.5-pro" });

async function analyzeWithCustomPrompt(
  imagePath: string,
  prompt: string
): Promise<string> {
  const imageBuffer = fs.readFileSync(imagePath);
  const base64Image = imageBuffer.toString("base64");

  const result = await model.generateContent([
    {
      inlineData: {
        mimeType: "image/jpeg",
        data: base64Image,
      },
    },
    prompt,
  ]);

  const response = await result.response;
  return response.text();
}

// 使用例
const prompts = [
  "画像内の人物の人数を教えてください。",
  "画像に写っている食べ物の名前をすべて挙げてください。",
  "この画像が屋内か屋外かを判定してください。",
];

async function main() {
  for (const prompt of prompts) {
    const result = await analyzeWithCustomPrompt("./scene.jpg", prompt);
    console.log(`プロンプト: ${prompt}`);
    console.log(`結果: ${result}\n`);
  }
}

main().catch(console.error);
```

## 5. ハマりやすいポイント

### APIキーの管理
環境変数にAPIキーを設定する際、`.env`ファイルを使う場合は`dotenv`パッケージを忘れずに読み込んでください。Gitにコミットしないよう注意が必要です。

### 画像サイズの制限
Gemini 1.5 Proは1リクエストあたりの画像サイズに制限があります（最大20MB程度）。大きな画像は事前にリサイズするか、JPEG品質を下げてから送信してください。

### Base64エンコードのオーバーヘッド
ファイルサイズが大きい場合、Base64エンコードによりデータサイズが約33%増加します。APIのレイテンシに影響するため、画像は必要最低限の品質・サイズに調整することを推奨します。

### プロンプトの設計
「JSONで出力してください」と指示しても、Geminiが必ずしも厳密なJSONを返すとは限りません。パースが失敗した場合のフォールバック処理を実装しておくと安全です。

## 6. まとめ

Gemini APIとTypeScriptを組み合わせることで、画像解析のパイプラインを簡潔に実装できました。ポイントは以下の通りです。

- Base64エンコードした画像データをインラインで送信する
- `Promise.allSettled`でエラーハンドリングをしながらバッチ処理する
- プロンプトを動的に変更して解析内容を制御する

このパイプラインをベースに、画像の前処理（リサイズ・フォーマット変換）や結果のデータベース保存を追加すれば、実用的な画像解析システムが構築できます。ぜひ、自分のプロダクトに合わせてカスタマイズしてみてください。

---

**あわせて読みたい**

AIエージェント開発のさらに深い実践知については、こちらのnote記事で詳しく解説しています。実務での自動化やチーム開発のヒントになれば幸いです。

[AIエージェントに1ヶ月「収益化」を任せた結果：累計879円の全内訳と、5つの誤算](https://note.com/agent_workshop/n/n381aea9a90da)