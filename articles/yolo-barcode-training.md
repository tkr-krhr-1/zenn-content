---
title: "【YOLO】物体検知モデルを構築してみた"
emoji: "🎯"
type: "tech"
topics: ["yolo", "ultralytics", "python", "物体検出", "機械学習"]
published: false
---

※本記事は、読者の方にとって分かりやすい構成となるようAIのサポート活用し、私自身の経験と裏取りを元に書き下ろしたものです。

## はじめに

前回の[入門編](https://zenn.dev/tkr_krhr/articles/yolo-object-detection-intro)では、YOLOの概要と学習済みモデルを使った推論方法について解説しました。

今回の記事では、**Google Colab** と **Roboflow** を活用して、**バーコードデータセットでモデルをトレーニングする**具体的な手順を解説します。

![alt text](/images/yolo-barcode-training/image0.png)

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

Roboflow Universeでバーコードを検索し、データセットを選択します。

今回は1.34k imagesのデータセットを選びました。

![Roboflow Universeでバーコードを検索](/images/yolo-barcode-training/image1.png)

DatasetからYOLOv8を選択します。

![データセットの概要とYOLOv8形式の選択](/images/yolo-barcode-training/image2.png)

Download dataset > Show download codeでContinueを押すと以下の画面が出ます。

![Roboflowのコードスニペット](/images/yolo-barcode-training/image3.png)

コピーをして、Google Colabのセルで実行します。

![Google Colabでのデータセットダウンロード](/images/yolo-barcode-training/image4.png)

これでデータセットの準備が完了しました。


## 3. 実践：トレーニングの実行

準備が整ったら、実際にトレーニングを開始します。

コードを実行する前に、Google Colabのランタイム設定でGPUを選択します。

![Google Colabのランタイム設定](/images/yolo-barcode-training/image5.png)

表示された画面で、T4 GPUを選択し保存します。

**※ T4 GPUとは？**
NVIDIAが開発した、AIの学習や推論に特化したGPUです。
Google Colabでは、この強力な計算機を**無料**（使用状況による制限あり）で活用できます。

普通のパソコンのCPUで学習させると数時間以上かかる計算でも、T4 GPUを使えば**数分〜数十分**で終わらせることが可能です。

### Weights & Biases (W&B) を使った可視化

トレーニングの進捗や、**GPUの使用率**などをリアルタイムで可視化するために、機械学習用のダッシュボードツールである「Weights & Biases (W&B)」を活用します。YOLOv8は標準でW&Bとの連携に対応しています。

学習を始める前に、あらかじめW&Bへのアカウント登録とAPIキーの取得が必要です。[Weights & Biasesの公式サイト](https://wandb.ai/)にアクセスしてアカウントを作成（またはログイン）し、アカウント設定ページ等からAPIキーをコピーしておきましょう。

APIキーが準備できたら、以下のコードの `"自身のAPIキー"` の部分を書き換えてセルを実行します。
これにより、ライブラリのインストールとW&Bへのログインが行われます。

```python
# 1. ライブラリのインストール（-U で最新版に更新しつつインストール）
!pip install -U wandb

# 2. W&Bへのログイン
import wandb
wandb.login(key="自身のAPIキー")

# 3. YOLOの設定でW&Bを有効化
!yolo settings wandb=True
```

### 学習の開始

ログインが完了したら、以下のコードで学習を実行します。

```python
from ultralytics import YOLO
import datetime

wandb.init(project="yolov8-custom-training", name="trial-1")

# 1. ベースとなる「脳みそ（モデル）」を読み込む
model = YOLO('yolov8n.pt')

# 2. 学習スタート
# 引数 `project` と `name` を追加することで、保存先を指定できます。
results = model.train(
    data=f'{dataset.location}/data.yaml', # Roboflowで取得した data.yaml のパスを指定
    epochs=10,                            # 学習する回数（データセット全体を何周するか）
    imgsz=640                             # 学習時の画像サイズ（640ピクセルが標準）
)

wandb.finish()
```

実行すると以下のように学習が進みます。

![Google Colabのランタイム設定](/images/yolo-barcode-training/image7.png)

出力される各項目は「AIが今どれくらい学習のステップを進めていて、どのくらい賢くなっているか」を表しており、それぞれの意味は以下の通りです。

- **Epoch（エポック）**: 学習の進捗。画像データを1周することを「1エポック」と呼びます。（例：`1/10`なら全10周中の1周目）
- **GPU_mem**: 学習に使用しているGPUメモリの消費量です。
- **box_loss**: 検知枠（バウンディングボックス）のズレ誤差です。数字が小さいほど対象物を正確な枠で囲めています。
- **cls_loss**: クラス推論（種類の識別）の誤差です。数字が小さいほど、別のものと勘違いせずに正しく見分けられています。
- **dfl_loss**: 枠の境界線の誤差です。数字が小さいほどフチがブレずに鮮明に捉えられています。
- **Instances**: その回の学習ステップに含まれている対象物の個数です。

**Loss（ロス）系の3つの指標（box_loss, cls_loss, dfl_loss）はすべてAIの「誤差」を表すため、学習が進むにつれて数字がどんどん小さくなっていくのが理想的です。**

学習がスタートすると、実行ログの中にW&Bのダッシュボードへのリンク（例: `View run at https://wandb.ai/...`）が表示されます。
リンク先にアクセスして「System」タブを開くと、T4 GPUが実際にどれくらい使われているか（GPU使用率やメモリ使用量）をリアルタイムのグラフで確認できます。

![Google Colabのランタイム設定](/images/yolo-barcode-training/image6.png)

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
# model.predictで物体検出を実行する
results = model.predict(
    source=0,    # カメラの映像を取得する
    show=True,   # プレビュー画面を表示する
    stream=True, # カメラの映像をリアルタイムで処理する
    conf=0.8     # 信頼度が80%以上のものだけ表示する
)

# このループがある間だけ、カメラ画面が表示され続けます
for r in results:
    # 検出されたバーコードの情報を取得
    if len(r.boxes) > 0:
        for box in r.boxes:
            conf = box.conf[0] # 信頼度の数字を取得
            print(f"【認識中】バーコードを発見しました！ 信頼度: {conf:.2f}")
```

実行するとカメラウィンドウが開き、バーコードを認識します。

![alt text](/images/yolo-barcode-training/image8.png)

精度を上げるにはepochsを上げたり、データセットの量や質を上げたりする必要があります。

## まとめ

Ultralytics YOLOを使えば、複雑なディープラーニングのコードを書くことなく、独自のデータセットで高性能なモデルを構築できます。

今後も物体検出についての記事執筆を予定しています！ぜひフォローしてお待ちください！
