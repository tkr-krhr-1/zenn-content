---
title: "GeminiでPCを自動操作する！次世代AIエージェントの作り方"
emoji: "🤖"
type: "tech"
topics: ["Gemini", "AI", "RPA", "Python", "deepdive"]
published: false
---

## tl;dr

- Gemini APIを使うと、PCの画面を認識して操作する「AIエージェント」を構築できる。
- 画面のスクリーンショットを「観察」し、実行すべきクリックや入力といった「行動」をモデルが推論する。
- この「観察→推論→行動」のループにより、Webブラウザの操作などを自動化できる。

## 導入

GoogleのGeminiは、テキストや画像など複数の情報を同時に理解できるマルチモーダルAIです。この能力を応用することで、単なる対話AIに留まらない、非常に強力なツールを開発できます。その一つが、今回紹介する「PC操作AIエージェント」です。

これは、AIが人間のようにPCの画面を見て、次に何をすべきかを判断し、マウスやキーボードを操作するというもの。従来のRPA（Robotic Process Automation）ツールが苦手としていた、画面レイアウトの変更などにも柔軟に対応できる可能性を秘めています。

本記事では、Gemini APIを使ってこの次世代の自動化技術を実装する方法について、公式ドキュメントを参考にしながら、その仕組みから具体的なコードまでを解説します。

- **ポイント:**
  - Geminiのマルチモーダル機能を活用したPC操作エージェントの仕組みを理解する。
  - 「観察-行動ループ」というエージェントの基本設計を学ぶ。
  - Pythonを使った具体的な実装方法の初歩を掴む。

## 事前準備

この記事の内容を試すには、以下の知識や環境が必要です。

- **必要な知識:**
  - Pythonの基本的な文法
  - Gemini API（または他のLLM API）の利用経験
- **環境:**
  - Python 3.9以上
  - Google AI Python SDK: `pip install -q google-generativeai`
  - スクリーンショットやGUI操作のためのライブラリ（例: `Pillow`, `pyautogui`）

## 背景・基礎知識

### AIエージェントと「観察-行動ループ」

AIエージェントとは、**環境を観測し、目標を達成するために自律的に行動するシステム**のことです。その最も基本的な動作原理が「観測-行動ループ（Observation-Action Loop）」または「ReAct（Reasoning and Acting）」アプローチです。

1.  **観測 (Observation)**: エージェントが現在の環境の状態を把握します。今回のケースでは、PCのスクリーンショットを撮ることに相当します。
2.  **推論 (Reasoning)**: 観測した情報と、与えられた最終目標に基づき、次に行うべき最も合理的な行動を決定します。この頭脳部分をGeminiが担当します。
3.  **行動 (Action)**: 推論によって決定された行動（例: 「ボタンをクリックする」「テキストを入力する」）を実際に実行します。

この3つのステップを繰り返すことで、エージェントは目標達成まで自律的にタスクを進めていきます。

### Geminiの役割

本手法におけるGeminiの役割は、ループの「推論」部分です。開発者は、現在のスクリーンショット（観測結果）と最終目標をGeminiに渡し、「次は何をすべきか？」と問いかけます。Geminiは画像とテキストを理解し、次に行うべき行動を `click(x, y)` や `type("text")` のような、プログラムで解釈しやすい形式（通常はJSON）で返します。

## 本論: GeminiによるPC操作の仕組み

GeminiにPC操作を正しく推論させるには、プロンプトの設計が非常に重要です。

### プロンプトの構成要素

1.  **システム命令 (System Instruction)**:
    エージェントの役割や能力、制約を定義します。「あなたはPCを操作するAIアシスタントです。画面を見て、指示された目標を達成してください。行動はJSON形式で出力してください」といった指示を与えます。

2.  **最終目標 (Goal)**:
    ユーザーがエージェントに達成してほしいタスクを具体的に伝えます。「Googleで明日の東京の天気を検索して」など。

3.  **観測結果 (Observation)**:
    現在のPC画面のスクリーンショット画像をプロンプトに含めます。これがAIの「目」になります。

4.  **出力形式の指定 (Output Schema)**:
    AIにどのような形式で行動を返してほしいかを厳密に定義します。JSONスキーマを用いることで、`"action": "click"` や `"action": "type"` といったキーと、それに対応する値（座標や入力テキスト）を安定して出力させることができます。

### Pythonによる実装フロー

1.  **スクリーンショットの取得**: `Pillow` などのライブラリを使い、現在の画面を画像として取得します。
2.  **プロンプトの組み立て**: 上記の4つの要素（システム命令、目標、画像、出力形式）を組み合わせて、Gemini APIに送信するプロンプトを作成します。
3.  **Gemini APIの呼び出し**: `google-generativeai` SDKを使い、組み立てたプロンプトをGeminiモデル（例: `gemini-1.5-pro`）に送信します。
4.  **応答の解析**: Geminiから返ってきたJSON形式の応答をパースし、`action`の種類（`click`, `type`など）とパラメータを取り出します。
5.  **アクションの実行**: `pyautogui` などのGUI操作ライブラリを使い、解析したアクション（マウスクリックやキーボード入力）を実際に実行します。
6.  **ループ**: 目標が達成されるまで、ステップ1〜5を繰り返します。

## 具体例・コード例

以下は、「メモ帳アプリを開き、'Hello, Gemini!'と入力する」タスクの擬似コードです。

```python
import google.generativeai as genai
from PIL import ImageGrab
import pyautogui
import json
import time

# Gemini APIキーを設定
# genai.configure(api_key="YOUR_API_KEY")

# Geminiモデルを初期化
# model = genai.GenerativeModel('gemini-1.5-pro-latest')

def run_agent_loop(goal: str):
    prompt = f"""
    あなたはPCを操作するAIアシスタントです。
    現在のスクリーンショットを見て、以下の目標を達成するための次の行動をJSONで出力してください。
    行動には 'click', 'type', 'scroll', 'wait', 'finish' があります。
    
    目標: {goal}
    """
    
    for i in range(5): # 無限ループを避けるための上限
        print(f"--- Step {i+1} ---")
        
        # 1. 観測: スクリーンショットを撮る
        screenshot = ImageGrab.grab()
        
        # 2. 推論: Geminiに行動を決定させる
        # response = model.generate_content([prompt, screenshot])
        # 以下はダミーの応答
        if i == 0:
            response_text = '{"action": "click", "comment": "メモ帳のアイコンをクリック", "x": 100, "y": 200}'
        elif i == 1:
            response_text = '{"action": "wait", "duration": 2}'
        else:
            response_text = '{"action": "type", "text": "Hello, Gemini!"}'

        print(f"AIの推論: {response_text}")
        action_data = json.loads(response_text)
        
        # 4. 行動: 推論されたアクションを実行
        action_type = action_data.get("action")
        
        if action_type == "click":
            pyautogui.click(action_data["x"], action_data["y"])
        elif action_type == "type":
            pyautogui.write(action_data["text"])
        elif action_type == "wait":
            time.sleep(action_data["duration"])
        elif action_type == "finish":
            print("目標達成！")
            break
            
        time.sleep(1)

# エージェントを実行
# run_agent_loop("メモ帳を開いて 'Hello, Gemini!' と入力する")
```
*注意: このコードは概念を示すためのものであり、実際にはAPIキーの設定や、より堅牢なプロンプト、エラーハンドリングが必要です。*

## 応用・発展

- **複雑なWeb操作の自動化**: 複数ページにまたがるECサイトでの商品購入や、SNSへの自動投稿など。
- **GUIアプリのテスト**: 特定のアプリケーションのボタンを順番にクリックし、表示が正しいかを確認するテストの自動化。
- **データ収集**: 複数のWebサイトから特定の情報を収集し、Excelなどにまとめる作業の自動化。

## まとめ・今後の展望

本記事では、Gemini APIを用いてPC操作を自動化するAIエージェントの基本的な仕組みと実装方法を解説しました。この技術はまだ発展途上ですが、将来的には、自然言語で指示するだけであらゆるPC作業を代行してくれる、真のデジタルアシスタントが実現するかもしれません。

一方で、意図しない操作を実行してしまうセキュリティリスクも存在するため、実際の運用ではサンドボックス環境で実行するなどの安全対策が不可欠です。

AIの「目」と「頭脳」を手に入れた自動化技術が、私たちの働き方をどのように変えていくのか、今後の発展から目が離せません。

## 参考リンク

-   [公式ドキュメント: コンピュータの使用 | Gemini API](https://ai.google.dev/gemini-api/docs/computer-use?hl=ja)
-   [google-generativeai (Python SDK) on GitHub](https://github.com/google/generative-ai-python)
-   [PyAutoGUI Documentation](https://pyautogui.readthedocs.io/)