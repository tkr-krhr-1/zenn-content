---
title: "Google AntigravityのSkills機能で開発を自動化しよう！"
emoji: "🚀"
type: "tech"
topics: ["antigravity", "ai", "devtools"]
published: false
---

# はじめに
こんにちは！今回はGoogleの新IDE「Antigravity」に追加された**Skills機能**について解説します。
これを使うと、いつものルーチンワークをAIエージェントに任せることができるようになります。

# Skillsとは？
簡単に言うと**「AIエージェントに渡す手順書」**です。
`SKILL.md`というファイルを置くだけで、エージェントがその手順を学習し、実行してくれるようになります。

:::message
**補足**
AnthropicのClaude Codeなどが採用している「Agent Skills」標準と同じフォーマットです。
:::

# 実践：挨拶スキルの作成
まずは簡単な挨拶スキルを作ってみましょう。

## ディレクトリ構成
```text
.agent/
  └── skills/
      └── greeter/
          └── SKILL.md
SKILL.mdの記述
---
name: greeter
description: ユーザーに元気に挨拶するスキル
---
# Greeter Skill
ユーザーが挨拶したら、全力で元気に返事してください！
まとめ
Skills機能を使いこなせば、自分だけの最強エージェントが作れます。ぜひ試してみてください！