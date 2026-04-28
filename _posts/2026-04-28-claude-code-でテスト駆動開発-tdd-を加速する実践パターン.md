---
layout: post
title: "Claude Code でテスト駆動開発（TDD）を加速する実践パターン"
date: 2026-04-28 09:00:00 +0900
categories: [ClaudeCode, TDD]
tags: [ClaudeCode, TDD, テスト, TypeScript, 開発効率]
description: Claude Code でテスト駆動開発（TDD）を加速する実践パターンの実装方法を解説します。
---
# Claude Code でテスト駆動開発（TDD）を加速する実践パターン

Claude Codeを使っていると、実装そのものより「どこから直し始めるか」「変更を広げすぎないか」のほうが難しい場面があります。特にTypeScriptやJavaScriptでは、勢いで修正すると動いたように見えて別の箇所を壊しやすいです。

そこで相性がいいのがTDDです。Claude Codeはコード生成だけでなく、既存コードの読み取り、失敗テストの原因整理、最小修正の提案までまとめて進められるので、テストを先に置く流れと噛み合います。この記事では、中級エンジニア向けに、Claude CodeでTDDを速く回すための実践パターンを整理します。

## 先に「失敗するテスト」を固定する

Claude Codeに最初から実装を書かせるより、先に失敗するテストを置いたほうがブレにくいです。依頼も小さくできます。

たとえば文字列の空白を正規化する関数を追加したいなら、最初にこう置きます。

```ts
// normalize-space.test.ts
import { describe, expect, test } from "vitest";
import { normalizeSpace } from "./normalize-space";

describe("normalizeSpace", () => {
  test("連続する空白を1つにまとめる", () => {
    expect(normalizeSpace("foo   bar")).toBe("foo bar");
  });

  test("前後の空白を取り除く", () => {
    expect(normalizeSpace("  hello world  ")).toBe("hello world");
  });
});
```

この状態でClaude Codeには、実装を丸投げするより次のように頼むほうが安定します。

```text
このテストを通す最小実装だけ追加して。既存APIは増やさず、副作用も入れないで。
```

TDDで重要なのは、AIに自由度を与えすぎないことです。テストが仕様になっていれば、実装が大きく逸れにくくなります。

## Red-Green-Refactorを会話でも分ける

Claude Codeは一度にたくさん進められますが、TDDでは段階を混ぜないほうがよいです。実務では、次の3段階に分けると扱いやすくなります。

1. Red: 失敗テストを追加する
2. Green: そのテストだけ通す最小修正を入れる
3. Refactor: 重複や命名を整える

たとえば `normalizeSpace` の実装は、Green段階ならこれで十分です。

```ts
// normalize-space.ts
export function normalizeSpace(input: string): string {
  return input.trim().replace(/\s+/g, " ");
}
```

ここで大事なのは、Greenの段階で「ついでの最適化」をさせないことです。Claude Codeにありがちな広めの修正を避けるには、依頼文も段階ごとに切ったほうがよいです。

```text
今はGreenだけやって。実装を通すのに不要なリファクタはしないで。
```

この一文だけでも、不要な書き換えはかなり減ります。

## バグ修正でも、再現テストを先に作る

TDDは新規実装だけでなく、不具合修正でも効きます。むしろClaude Codeと組み合わせるなら、既存バグの再現テストづくりが一番実用的です。

たとえば合計金額の計算で負の値を許してしまう不具合があったとします。先に再現テストを置きます。

```ts
// total-price.test.ts
import { expect, test } from "vitest";
import { calcTotalPrice } from "./total-price";

test("負の金額が含まれるときは例外を投げる", () => {
  expect(() => calcTotalPrice([1200, -300, 500])).toThrowError(
    "price must be non-negative",
  );
});
```

そのうえでClaude Codeには、原因調査と修正を分けて頼みます。

```text
この失敗テストの原因候補を2つに絞って。関連ファイルを挙げてから、最小修正案を出して。
```

最初から「直して」だけだと、関数全体を書き直す方向へ寄りがちです。再現テストがあると、変更範囲を狭く保ちやすくなります。

## テスト実行は小さく回す

Claude CodeでTDDを速くするには、毎回フルスイートを回さないことも大事です。まずは対象テストだけを回し、区切りで全体確認するほうが効率がよいです。

```bash
npx vitest run normalize-space.test.ts
npx vitest run total-price.test.ts
npx vitest run
```

会話でも、最初から「全部のテストを見て」ではなく、「この1ファイルだけ見て」「この失敗だけ直して」と刻んだほうが安定します。Claude Codeは文脈を多く読めますが、TDDでは入力を絞ったほうが結果がよくなります。

## 実務で使いやすい運用パターン

最後に、Claude CodeでTDDを回すときに効きやすい運用を3つに絞ります。

1. 失敗テストを先に書いて仕様を固定する
2. RedとGreenとRefactorを別メッセージに分ける
3. バグ修正では再現テストを先に作る

Claude Codeは「全部考えて直す」使い方もできますが、TDDと組み合わせるなら「小さい失敗を先に定義して、そこに向けて最小修正を積む」ほうが強いです。

## まとめ

Claude CodeでTDDを加速するコツは、AIに大きな自由度を渡さず、失敗テストを先に置いて段階ごとに依頼することです。特にTypeScriptやJavaScriptでは、再現テストを先に作るだけで修正の質がかなり安定します。

新機能でもバグ修正でも、まずは1本の失敗テストから始めるのが現実的です。Claude Codeは実装を速くしますが、TDDはその速さを壊さないためのガードとして機能します。
