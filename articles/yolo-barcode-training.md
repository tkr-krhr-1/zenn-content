---
title: "YOLOv8 × Roboflow × Colab：バーコード検知モデルを構築する"
emoji: "🎯"
type: "tech"
topics: ["yolo", "ultralytics", "python", "物体検出", "機械学習"]
published: false
---

※本記事は、読者の方にとって分かりやすい構成となるようAIのサポート活用し、私自身の経験と裏取りを元に書き下ろしたものです。

## はじめに

前回の[入門編](https://zenn.dev/tkr_krhr/articles/yolo-object-detection-intro)では、YOLOの概要と学習済みモデルを使った推論方法について解説しました。

今回の記事では、**Google Colab** と **Roboflow** を活用して、**バーコードデータセットでモデルをトレーニングする**具体的な手順を解説します。

## 1. YOLOでトレーニングを行うメリット

### コード記述量の少なさ
PyTorchでは、データの読み込み、モデル定義、損失関数の設定、学習ループの記述などに数百行のコードが必要です。
YOLOはこれらが**パッケージ化**されているため、数行のコマンドや関数を呼び出すだけで学習が完結します。

### 物体検出に特化した設定
物体検出特有の複雑な処理を、開発者が意識しなくて済むよう最適化されています。

PyTorch: 座標計算やNMS（重複検出の除去）などを自前で実装・調整する必要がある。

YOLO: 最新のデータ拡張、ハイパーパラメータの自動調整、GPU設定などがデフォルトで最適化されており、何もしなくても高い精度が出やすい。

### 動作環境・推論へのスムーズな移行
学習が終わった後の動かすステップへのハードルが低いです。

PyTorch: モデルの保存形式や、推論用の前処理コードを別途管理する必要がある。

YOLO: 学習済みファイルをそのまま使い、iOS、Android、Webブラウザ、組み込み機器へ変換・書き出しするためのツールが標準装備されています。


## 2. データセットの準備

### データセットの取得（Roboflow）


Roboflowにアクセスして、データセットを用意します。※登録が必要です。
今回は、公開されているバーコードのデータセットを使用します。

![Roboflow Universeでバーコードを検索](/images/yolo-barcode-training/roboflow-universe-search.png)

![データセットの概要とYOLOv8形式の選択](/images/yolo-barcode-training/roboflow-dataset-overview.png)

Download dataset > Show download codeでContinueを押すと以下の画面が出ます。

![Roboflowのコードスニペット](/images/yolo-barcode-training/roboflow-code-snippet.png)

コピーをして、Google Colabのセルで実行します。

![Google Colabでのデータセットダウンロード](/images/yolo-barcode-training/image.png)

これでデータセットの準備が完了しました。


## 3. 実践：トレーニングの実行

準備が整ったら、実際にトレーニングを開始します。

コードを実行する前に、Google Colabのランタイム設定でGPUを選択します。

![Google Colabのランタイム設定](/images/yolo-barcode-training/google-colab-runtime-settings.png)

表示された画面で、T4 GPUを選択し保存します。

**※ T4 GPUとは？**
NVIDIAが開発した、AIの学習や推論に特化したGPUです。
Google Colabでは、この強力な計算機を**無料**（使用状況による制限あり）で活用できます。

普通のパソコンのCPUで学習させると数時間以上かかる計算でも、T4 GPUを使えば**数分〜数十分**で終わらせることが可能です。

### Weights & Biases (W&B) を使った可視化

トレーニングの進捗や、**GPUの使用率**などをリアルタイムで可視化するために、機械学習用のダッシュボードツールである「Weights & Biases (W&B)」を活用します。YOLOv8は標準でW&Bとの連携に対応しています。

学習を始める前に、以下のコードを実行してライブラリのインストールとW&Bへのログインを行います。

```python
# 必要なライブラリのインストール
!pip install ultralytics wandb

import wandb
# W&Bにログイン
wandb.login()
```

セルを実行するとログイン手順が表示されます。記載されているリンクからW&Bのアカウントを作成（またはログイン）し、指示に従ってAPIキーをコピーしてColabの入力欄に貼り付け、Enterキーを押してください。

### 学習の開始

ログインが完了したら、以下のコードで学習を実行します。

```python
from ultralytics import YOLO

# 1. ベースとなる「脳みそ（モデル）」を読み込む
# 最初から賢いyolov8n（Nanoサイズ：一番軽くて速い）を使います
model = YOLO('yolov8n.pt')

# 2. 学習スタート！
results = model.train(data=f'{dataset.location}/data.yaml', epochs=1, imgsz=640)
# data: Roboflowで取得した data.yaml のパスを指定
# epochs: 学習する回数。まずは1回で様子を見ます
# imgsz: 学習時の画像サイズ。640ピクセルが標準
```

学習がスタートすると、実行ログの中にW&Bのダッシュボードへのリンク（例: `View run at https://wandb.ai/...`）が表示されます。
リンク先にアクセスして「System」タブを開くと、T4 GPUが実際にどれくらい使われているか（GPU使用率やメモリ使用量）をリアルタイムのグラフで確認できます。

### 学習済みモデルの検証

学習が完了すると、`runs/detect/train/weights/best.pt` にモデルが保存されます。
これを以下のコードで手元のPCにダウンロードします。

```python
from google.colab import files

# best.pt をローカルPCにダウンロード
files.download('runs/detect/train/weights/best.pt')
```

ダウンロードした best.pt を、手元のPCで作成したディレクトリに移動させます。

同じ階層にモデルで物体検出するために以下のコードでファイルを作成します。

```python
from ultralytics import YOLO

# 1. モデルの読み込み
model = YOLO('best.pt')

# 2. カメラの起動
results = model.predict(source=0, show=True, stream=True, conf=0.1)
# show=True がプレビュー画面を表示する設定です

# このループがある間だけ、カメラ画面が表示され続けます
for r in results:
    # 検出されたバーコードの情報を取得
    if len(r.boxes) > 0:
        for box in r.boxes:
            conf = box.conf[0] # 信頼度（どのくらいバーコードっぽいか）
            print(f"【認識中】バーコードを発見しました！ 信頼度: {conf:.2f}")
```

実行するとカメラウィンドウが開き、バーコードを認識します。
※信頼度を0に設定しているため、テーブルもバーコードと検知されています。

![alt text](/images/yolo-barcode-training/yolo-detection-result.png)

学習回数が一回なので、精度は甘いですが、回数を重ねれば、より高精度なモデルを構築できます。


## まとめ

Ultralytics YOLOを使えば、複雑なディープラーニングのコードを書くことなく、独自のデータセットで高性能なモデルを構築できます。

今後もYOLOについての記事執筆を予定しています！ぜひフォローしてお待ちください！
