---
title: "【Google Antigravity】「Skills」機能を試してみた"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googleantigravity", "ai", "agent"]
published: false
---

## 🚀 はじめに

Google Antigravity、使っていますか？ 最近、新機能として **Skills（スキル）** が追加されました。

これまでもカスタム指示（Customizations/Rules）でエージェントの挙動を調整できましたが、Skills はどうやら「もっと高度なタスク」を「必要な時だけ」実行させるための機能のようです。

「これは開発フローが変わるかもしれない...」と思い、実際に試してみました。 今回は、実用的な**「自動コードレビュースキル」**の作成を通して、その使い勝手と Customizations との違いを解説します。

## 📚 概要: Skills（スキル）とは？

Skills は、**エージェントに特定のタスクの進め方やベストプラクティスを教えるための「再利用可能なパッケージ」**です。

単なるプロンプトテンプレートではありません。以下の 4 つの要素を組み合わせることで、複雑な作業を自律的に行わせることができます。

- **SKILL.md**: 指示書（いつ、どう動くか）
- **scripts/**: 道具（Python スクリプトなど、エージェントが実行するツール）
- **resources/**: 素材（チェックリスト、テンプレート、社内データ）
- **examples/**: お手本（理想的な出力例、コード規約）

これらをプロジェクトの `.agent/skills/` ディレクトリに置くだけで、エージェントは文脈に応じて勝手にスキルを使ってくれるようになります。

### なぜ Skills を使うのか？（3 つのメリット）

Customizations で長大な指示を書くのではなく、Skills として切り出すことには明確なメリットがあります。

#### 1. コンテキストの節約と「集中力」の維持

Customizations にすべての業務マニュアルを詰め込むと、常に大量のトークンを消費するだけでなく、エージェントが指示の優先順位を見失う（ハルシネーションを起こす）原因になります。
Skills は**必要な時だけ読み込まれる（Dynamic Loading）**ため、エージェントは目の前のタスクに集中でき、回答の精度が向上します。

#### 2. 「確率」と「ロジック」のいいとこ取り

LLM は「滑らかな文章」を作るのは得意ですが、「文字数を正確に数える」「複雑な計算をする」といった厳密な処理は苦手です。
Skills を使えば、**厳密なチェックは `scripts/`（Python 等）に任せ、その結果を人間らしく伝える部分を LLM が担う**という、分業体制が構築できます。

#### 3. Git でのバージョン管理と共有

Skills はフォルダとファイルの実体として存在するため、Git で管理できます。
「新人メンバーが入ったら、とりあえずこの Skills フォルダを `git pull` しておいて」と言うだけで、チーム全員が同じ品質基準（Lint ルールやレビュー観点）を即座に共有できます。

## 🛠️ 準備

今回は検証として、以下の挙動をする `code-reviewer` スキルを作ります。

1. コードを受け取ったら、まずスクリプトで機械的なミス（TODO 残存など）をチェックする。
2. チェックリストを参照し、セキュリティやパフォーマンスの観点漏れを防ぐ。
3. お手本に従って、優しく建設的なトーンで指摘する。

### フォルダ構成

プロジェクトルート（またはグローバル設定）に以下のディレクトリとファイルを作成します。Skills の面白い点は、フォルダ構成そのものがエージェントへの入力になる点です。

```plaintext
.agent/skills/code-reviewer/
├── SKILL.md                  # メインの指示書
├── scripts/
│   └── simple_linter.py      # 簡易チェックツール
├── resources/
│   └── review_checklist.md   # レビュー観点リスト
└── examples/
    └── review_style.md       # フィードバックの書き方見本
```

それでは実際にファイルの中身を作っていきましょう。

### 1. 指示書 (SKILL.md)

エージェントへの「指令」です。ここで各リソースを紐付けます。YAML フロントマターで名前と説明を定義します。

:::details SKILL.md

```markdown
---
name: code-reviewer
description: コードの品質、安全性、可読性をチェックし、改善案を提示するスキル。PRレビューやリファクタリング相談時に使用。
---

# Code Reviewer Skill

提示されたコードに対してレビューを行い、改善点を指摘してください。

## レビュー手順

1. **自動チェックの実行**

   - 対象のコードファイルがある場合、以下のスクリプトを実行して基本的な問題をスキャンしてください。
   - コマンド: `python scripts/simple_linter.py <ファイルパス>`
   - コードがチャットに直接貼り付けられただけの場合は、このステップをスキップして目視確認してください。

2. **観点の確認**

   - `resources/review_checklist.md` を読み込み、各項目（セキュリティ、パフォーマンス等）についてコードを確認してください。

3. **フィードバックの作成**
   - `examples/review_style.md` のトーン＆マナー（口調・書き方）を参照してください。
   - 以下の構成で回答してください：
     1. **全体的な感想**: コードの意図を理解したポジティブなコメント。
     2. **重要課題**: バグやセキュリティリスクなど、必ず直すべき点。
     3. **改善提案**: 可読性向上や最適化のための提案（任意）。
     4. **修正コード例**: 具体的にどう直すべきかの Diff やコードブロック。

## 注意点

- 批判だけをせず、なぜ修正が必要なのか（Why）を説明してください。
- 変数名やコメントの不足についても積極的に指摘してください。
```

:::

### 2. 道具 (scripts/simple_linter.py)

LLM だけでレビューすると見落としがちな単純な文字列マッチングは、従来のスクリプト（道具）に任せます。

:::details simple_linter.py

```python
import sys
import re

def analyze_code(file_path):
    """
    コード内の怪しいパターンを簡易検出するスクリプト
    """
    patterns = {
        "FIXME/TODO": r"(TODO|FIXME)",
        "Print Statement": r"(print\(|console\.log\()",
        "Hardcoded Token": r"(token|password|secret)\s*=",
        "Generic Exception": r"except Exception:",
    }

    print(f"--- Analyzing {file_path} ---")

    issues_found = False
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            lines = f.readlines()

        for i, line in enumerate(lines):
            for issue_name, pattern in patterns.items():
                if re.search(pattern, line):
                    # print文などは意図的な場合もあるのでWARNレベル
                    level = "WARN" if "Print" in issue_name else "review-required"
                    print(f"Line {i+1} [{level}]: found {issue_name} -> {line.strip()}")
                    issues_found = True

        if not issues_found:
            print("No obvious issues found by simple_linter.")

    except FileNotFoundError:
        print("Error: File not found.")

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python simple_linter.py <file_path>")
    else:
        analyze_code(sys.argv[1])
```

:::

### 3. 素材 (resources/review_checklist.md)

レビューの「質」を担保するためのカンニングペーパーです。ここに社内のセキュリティ基準などを書いておくと強力です。

:::details review_checklist.md

```markdown
# コードレビュー・チェックリスト

レビュー時は以下の観点を必ず確認してください。

## 🛡️ 安全性 (Security)

- SQL インジェクションや XSS の脆弱性はないか？
- パスワードや API キーがハードコーディングされていないか？
- 入力値の検証（バリデーション）は適切か？

## 🚀 パフォーマンス (Performance)

- ループの中で重い処理（DB 接続など）を行っていないか？
- 無駄なメモリアロケーションはないか？
- N+1 問題が発生する可能性はないか？

## 📖 可読性・保守性 (Readability)

- 変数名・関数名は「何をするか」を表しているか？（`data`, `temp` などの曖昧な名前は NG）
- 複雑なロジックにはコメントがついているか？
- 関数が長すぎないか（単一責任の原則）？
- エラーハンドリングは適切か（エラーを握りつぶしていないか）？

## 🧪 テスト (Testing)

- エッジケース（境界値、null、空リストなど）は考慮されているか？
```

:::

### 4. お手本 (examples/review_style.md)

エージェントの「口調」や「指摘フォーマット」を制御します。これがあるだけで「AI っぽい回答」が「シニアエンジニアっぽい回答」に変わります。

:::details review_style.md

````markdown
# 良いレビューの書き方例

## 悪い例 ❌

> 10 行目の変数名がわかりにくいです。直してください。あとループが無駄です。

## 良い例 ✅

### 全体的な感想

機能の実装ありがとうございます！ロジックは明確でよく動いているようです。いくつか保守性とパフォーマンスの観点で提案があります。

### 改善提案

**1. 変数名の具体化 (Line 10)**
`d` という変数名だと、後から見た人が何のデータかわかりにくいです。格納されているのが日付データなら `target_date`、辞書なら `user_profile` などに変更することをお勧めします。

**2. ループ内での計算 (Line 15-20)**
ループの中で毎回 `calculate_tax_rate()` を呼んでいますが、税率は固定値のようなのでループの外に出すことでパフォーマンスが向上します。

```python
# 修正案
tax_rate = calculate_tax_rate()  # ループ外へ移動
for item in items:
    price = item.price * tax_rate
    ...
```

**3. エラーハンドリング**
`except Exception:` で全てのエラーをキャッチしていますが、予期せぬバグを見逃す可能性があります。`ValueError` など、想定されるエラーのみをキャッチするように変更しましょう。
````

:::

## 💻 実践：エージェントにレビューさせてみた

ファイルを配置後、実際に開発中のコード（経費精算機能）のレビューを依頼してみました。 こちらからは「スキルを使って」とは言わず、シンプルに一言だけ伝えます。

- **プロンプト**: 「コードレビューして」

### 1. スキルの自動発動とスクリプト実行

エージェントは即座に意図を理解し、思考プロセス（Thought）の中で `code-reviewer` スキルを起動しました。 注目すべきは、自ら `simple_linter.py` を実行して機械的なチェックを行っている点です。

![alt text](/images/antigravity-skills/image-1.png)

### 2. 生成されたレビュー結果

スクリプトの実行結果と、`review_checklist.md` の観点を組み合わせ、期待通りの回答が生成されました。

![alt text](/images/antigravity-skills/image-2.png)
![alt text](/images/antigravity-skills/image-3.png)
![alt text](/images/antigravity-skills/image-4.png)



## 💡 他に使えそうなアイデア

Code Reviewer 以外にも、工夫次第で様々なスキルが作れそうです。

#### 1. テストコード生成 (test-generator)

- **Point**: `examples/` に「自社プロジェクトのテストの書き方（pytest の fixture の使い方など）」を置いておく。
- **効果**: 誰が書いても（AI に書かせても）同じ作法のテストコードが生成されるようになります。

#### 2. SQL クエリ作成 (sql-expert)

- **Point**: `resources/` に `schema.md`（テーブル定義書） を置いておく。
- **効果**: 「先月の売上出して」と言うだけで、正しいテーブル結合を行った SQL が一発で出力されます。いちいちスキーマを教える必要がなくなります。

#### 3. バグ報告書作成 (bug-reporter)

- **Point**: `resources/` にバグ報告の Markdown テンプレートを置く。`scripts/` にログ収集スクリプトを置く。
- **効果**: エラーが起きた時に「報告書作って」と言うだけで、ログ付きの綺麗なチケット文章が完成します。

## 最後に

このアカウントでは、明日から開発フローを変えるための実践方法を共有していきます！
更新を見逃さないよう、ぜひ今のうちに Zenn のアカウントをフォローしてお待ちください！！！
