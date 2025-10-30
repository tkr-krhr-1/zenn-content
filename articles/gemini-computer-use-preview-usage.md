---
title: "Geminiでブラウザを自動操作！Computer Use Previewの使い方"
emoji: "🤖"
type: "tech"
topics: ["Gemini", "AI", "RPA", "Python", "Playwright"]
published: false
---

## tl;dr

- GoogleのGemini APIを利用して、自然言語でブラウザ操作を自動化するAIエージェント「Computer Use Preview」のセットアップ方法を解説。
- `playwright`をベースにしており、Chromeブラウザを自動で動かすことができる。
- 環境変数を設定し、`main.py`を実行するだけで、指定したクエリ（指示）に基づいたブラウザ操作が可能になる。

## 導入

Googleが提供するGeminiは、テキストだけでなく多様なモーダルを扱える強力なAIモデルです。その応用範囲は広く、単なる対話に留まりません。今回紹介する「Computer Use Preview」は、Geminiの能力を活用し、自然言語の指示だけでウェブブラウザを操作できるAIエージェントです。

例えば、「Googleで今日の天気を調べて」と指示するだけで、AIが自律的にブラウザを立ち上げ、検索を実行します。この記事では、このツールを自身のPCで動かすための、具体的なセットアップ手順から実行方法までを詳しく解説します。

- **ポイント:**
  - Geminiを活用したブラウザ操作エージェントの概要を理解する。
  - Computer Use Previewのインストールと設定手順を学ぶ。
  - 実際に自然言語でブラウザを操作する方法を把握する。

## 事前準備

この記事の内容を試すには、以下の知識や環境が必要です。

- **必要な知識:**
  - Pythonとpipの基本的な操作
  - コマンドライン（ターミナル）の基本的な使い方
  - Gitの基本的な操作
- **環境:**
  - Python 3.9以上
  - Gitがインストールされていること
  - Gemini APIキー（取得方法は[こちら](https://ai.google.dev/gemini-api/docs/api-key?hl=ja)）

## 背景・基礎知識

### Computer Use Previewとは

Gemini 2.5 Computer Useは、大規模言語モデル（LLM）であるGeminiを利用して、ユーザーの自然言語による指示を具体的なブラウザ操作に変換し、実行するAIエージェントです。内部では、ブラウザ自動化ツールである`Playwright`が使われており、LLMが「次に何をすべきか」を推論し、Playwrightがその指示に従ってブラウザを動かす、という仕組みになっています。

### Playwright

Playwrightは、Microsoftが開発するモダンなブラウザ自動化ライブラリです。Chromium, Firefox, WebKitといった主要なブラウザエンジンに対応しており、単一のAPIでクロスブラウザのテストや操作が可能です。本ツールでは、このPlaywrightを介してChromeブラウザを操作します。

- **ポイント:**
  - Geminiが「頭脳」として指示を出し、Playwrightが「手足」としてブラウザを動かす。
  - 従来のRPAツールより柔軟な操作が期待できる。

## 本論：セットアップと実行手順

### 1. リポジトリのクローン

まず、公式のGitHubリポジトリからソースコードをクローンします。

```shell
git clone https://github.com/google-gemini/computer-use-preview.git
cd computer-use-preview
```

### 2. 依存ライブラリのインストール

次に、Pythonの仮想環境を作成し、必要なライブラリをインストールします。

```shell
# 仮想環境の作成と有効化
python3 -m venv .venv
source .venv/bin/activate

# 依存ライブラリのインストール
pip install -r requirements.txt
```

### 3. Playwrightとブラウザのインストール

Playwrightのコマンドを使い、操作に必要なブラウザ（Chrome）をインストールします。

```shell
playwright install
```

### 4. APIキーの設定

Gemini APIを利用するために、取得したAPIキーを環境変数に設定します。

```shell
export GEMINI_API_KEY="YOUR_API_KEY"
```
*`.bashrc`や`.zshrc`に追記しておくと便利です。*

### 5. エージェントの実行

すべての準備が整ったら、`main.py`スクリプトを実行してエージェントを起動します。`--query`引数に、実行させたい操作を自然言語で渡します。

```shell
python3 main.py --query "Googleにアクセスして'Hello World'と検索して"
```

このコマンドを実行すると、新しいChromeウィンドウが立ち上がり、AIが自動でGoogleにアクセスして検索を行う様子が確認できます。

- **ポイント:**
  - 仮想環境を使うことで、プロジェクトの依存関係をクリーンに保つ。
  - `playwright install`で、操作対象のブラウザが自動でセットアップされる。
  - 操作内容は`--query`引数で自由に指定できる。

## 具体例・コード例

### GoogleでGeminiの公式サイトを検索する

```shell
python3 main.py --query "GoogleでGeminiの公式サイトを検索して"
```

### 競合ECサイトの売れ筋商品を検索する

```shell
python3 main.py --query "GoogleでAmazonを検索し、売れ筋商品を検索して"              
```



## 応用・発展

- **Vertex AIでの利用**:
  環境変数を設定することで、Gemini APIの代わりにGoogle CloudのVertex AIを利用することも可能です。
  ```shell
  export USE_VERTEXAI=true
  export VERTEXAI_PROJECT="your-gcp-project-id"
  export VERTEXAI_LOCATION="your-gcp-location"
  ```

- **複雑なタスクへの挑戦**:
  「特定のECサイトで商品を検索し、カートに入れる」といった、より複雑な複数ステップのタスクを指示してみるのも面白いでしょう。プロンプトを工夫することで、AIの操作精度を向上させることができます。

## まとめ・今後の展望

本記事では、Geminiを活用したブラウザ操作エージェント「Computer Use Preview」の導入方法を解説しました。自然言語でPC操作を自動化するこの技術は、まだプレビュー段階ですが、今後の発展によっては、私たちの定型業務を大きく変えるポテンシャルを秘めています。

現時点では、複雑な操作には対応しきれない場面もありますが、AIエージェント技術の進化は非常に速いです。ぜひこの機会に、次世代の自動化ツールに触れてみてください。

## 参考リンク

-   [google-gemini/computer-use-preview (GitHub)](https://github.com/google-gemini/computer-use-preview)
-   [Gemini API ドキュメント](https://ai.google.dev/gemini-api/docs?hl=ja)
-   [Playwright 公式サイト](https://playwright.dev/)
