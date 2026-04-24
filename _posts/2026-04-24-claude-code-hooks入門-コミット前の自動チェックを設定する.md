---
layout: post
title: "Claude Code Hooks入門：コミット前の自動チェックを設定する"
date: 2026-04-24 09:00:00 +0900
categories: [ClaudeCode, git]
tags: [ClaudeCode, git, 自動化, フック, 開発効率]
description: Claude Code Hooks入門：コミット前の自動チェックを設定するの実装方法を解説します。
---
# Claude Code Hooks入門：コミット前の自動チェックを設定する

Claude Codeを日常的に使い始めると、編集のたびに整形をかけたり、コミット前にlintやtestを回したりする作業がじわじわ効いてきます。毎回手でやると抜け漏れが出るので、先に自動化してしまったほうが安定します。

そのときに使えるのがHooksです。Claude CodeのHooksは、特定のイベントで任意のシェルコマンドを実行できる仕組みです。公式ドキュメントでも、編集後の自動整形、保護ファイルのブロック、通知などの用途が案内されています。中級エンジニアなら、まずは「編集直後の整形」と「コミット前チェック相当」の2段構えで入れるのが扱いやすいです。

## Hooksの基本を先に押さえる

Hooksは `~/.claude/settings.json` や `.claude/settings.json` に `hooks` キーを追加して設定します。プロジェクトで共有したいなら `.claude/settings.json`、自分だけの実験ならローカル設定に置くのが分かりやすいです。

Claude Codeには `PreToolUse` と `PostToolUse` など複数のイベントがあります。コミット前チェックに近い運用を作るなら、実際には「Claudeが編集した直後に毎回検査する」ほうが実務向きです。Gitの pre-commit に完全に同じではありませんが、問題をその場で返せるので修正が早くなります。

まずは編集後にPrettierを走らせる最小構成です。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"
          }
        ]
      }
    ]
  }
}
```

`PostToolUse` はツール成功後に走り、`Edit|Write` の matcher で編集系ツールのみを対象にできます。整形だけならこれで十分です。

## lintとtestを通して「コミット前に壊れていない状態」を保つ

次に、編集されたファイルがTypeScriptやJavaScriptなら `npm run lint` と `npm test` を実行する例です。毎回フルテストだと重いので、最初はlintだけ、もしくは関連パッケージだけに絞るのが現実的です。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "FILE=$(jq -r '.tool_input.file_path'); case \"$FILE\" in *.ts|*.tsx|*.js|*.jsx) npm run lint && npm test ;; esac"
          }
        ]
      }
    ]
  }
}
```

この形なら、対象外ファイルでは何もしません。Hooksは通常のシェルとして書けるので、チームの既存コマンドをそのまま流用できます。

ただし、設定ファイルに長いワンライナーを埋め込むと保守しにくくなります。少しでも複雑になるなら、スクリプトを分けたほうが安全です。

```bash
#!/bin/bash
set -eu

INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

case "$FILE_PATH" in
  *.ts|*.tsx|*.js|*.jsx)
    npm run lint
    npm test
    ;;
esac
```

このスクリプトを `.claude/hooks/check-changed-file.sh` に置いて、設定側から呼び出します。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/check-changed-file.sh"
          }
        ]
      }
    ]
  }
}
```

`$CLAUDE_PROJECT_DIR` を使うと、Claude Codeの起動位置に引きずられず、プロジェクト内スクリプトを安定して参照できます。

## 失敗させる基準を決める

Hooks運用で大事なのは、何を「止める条件」にするかです。公式リファレンスでは、終了コード2を返すとブロッキングエラーとして扱えます。つまり、危険なファイル編集や明らかなlint違反は止める、テストの一時失敗は通知だけにする、といった設計ができます。

最初から全部ブロックすると作業が重くなりがちです。おすすめは次の順番です。

1. `PostToolUse` で整形を自動化する
2. lintだけ実行して失敗時に気づけるようにする
3. 保護ファイルだけ `PreToolUse` でブロックする
4. テストは重さを見て対象を絞る

環境変数が必要なチェックを呼ぶ場合も、値そのものを設定ファイルに直接書かず、シェルや既存の実行環境から参照する形に寄せたほうが安全です。特に認証情報を含むチェックは、ログ出力に機密情報を混ぜないようにしておく必要があります。

## まとめ

Claude Code Hooksは、コミット前に人間がやっていた確認作業を、編集直後の自動チェックへ前倒しするのに向いています。最初は `PostToolUse` で Prettier と lint を回し、必要になったらスクリプト化して test や保護ルールを追加する、という段階導入が現実的です。

「pre-commit を完璧に再現する」より、「Claudeが変更した瞬間に壊れ方を見つける」ほうが運用効果は高いです。まずは軽いHooksから入れて、待ち時間と検出漏れのバランスを見ながら育てるのが失敗しにくいと思います。
\n\n---\n\n**あわせて読みたい**\n\nAIエージェント開発のさらに深い実践知については、こちらのnote記事で詳しく解説しています。実務での自動化やチーム開発のヒントになれば幸いです。\n\n[Claude・GPT・Geminiを「チーム」にしたら、1人で開発するより3倍速くなった話](https://note.com/kaiza/n/n9217c45f117e)