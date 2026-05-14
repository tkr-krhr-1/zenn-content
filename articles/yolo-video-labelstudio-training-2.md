---
title: "🧠動画データからYOLOモデルをゼロから作る——データセット準備・学習・評価編"
emoji: "⚙️"
type: "tech"
topics: ["yolo", "ultralytics", "python", "googlecolab", "物体検出"]
published: false
---

※本記事は、読者の方にとって分かりやすい構成となるようAIのサポートを活用し、私自身の経験と裏取りを元に書き下ろしたものです。

## 1. はじめに

[前回の記事（フレーム抽出・アノテーション編）](https://zenn.dev/tkr_krhr/articles/yolo-video-labelstudio-training)では、動画からフレームを抽出し、Label StudioでYOLO形式のアノテーションを行いました。

本記事ではその続きとして、アノテーション済みデータをYOLO学習用データセットに整形し、Google ColabのGPUで学習・評価するまでの工程を解説します。

### 使用技術スタック

| 役割 | ツール / ライブラリ |
|---|---|
| 物体検出フレームワーク | Ultralytics YOLOv8 |
| 学習・評価環境 | Google Colab（GPU） |
| データセット準備 | Python（shutil / pathlib） |

### 作業環境の全体像

| 章 | 作業内容 | 実行環境 |
|---|---|---|
| 2章 | Google Colab環境構築 | **Google Colab** |
| 3章 | データセット準備・アップロード | **ローカル** |
| 4章 | YOLOで学習 | **Google Colab（GPU）** |
| 5章 | モデルの評価 | **Google Colab（GPU）** |

### 記事の流れ

```
【ローカルPC】（前回記事の成果物）
YOLO形式ラベルファイル
  ↓ 3章：train/val/test分割 + data.yaml
YOLOデータセット（ZIP）→ Google Driveにアップロード

【Google Colab（GPU）】
  ↓ 4章：学習（NVIDIA T4 / A100）
best.pt（学習済みモデル）→ Google Driveに保存
  ↓ 5章：評価
評価指標（mAP / Precision / Recall）
```

## 2. Google Colab環境構築

4章の学習はGoogle Colabで実行します。新しいノートブックを開き、以下の手順でセットアップします。

#### GPUランタイムの有効化

「ランタイム」→「ランタイムのタイプを変更」で **T4 GPU** に設定します。

#### GPU接続の確認

```python
# Colabセル
import torch
print(torch.cuda.is_available())   # True であればGPU有効
print(torch.cuda.get_device_name(0))  # 例: Tesla T4
```

#### ライブラリのインストール

```python
# Colabセル
!pip install ultralytics -q
```

#### Google Driveのマウント

学習済みモデルをColabセッション終了後も保持するため、Google Driveをマウントします。

```python
# Colabセル
from google.colab import drive
drive.mount("/content/drive")
```

マウント後のパス例：`/content/drive/MyDrive/yolo_project/`

## 3. データセットの準備（ローカル）

### YOLOが要求するディレクトリ構成

YOLOは以下の構成を期待しています。`images/` と `labels/` が同じサブディレクトリ名でセットになっている必要があります。

```
dataset/
├── images/
│   ├── train/    # 学習用画像
│   ├── val/      # 検証用画像
│   └── test/     # テスト用画像
├── labels/
│   ├── train/    # 学習用ラベル
│   ├── val/      # 検証用ラベル
│   └── test/     # テスト用ラベル
└── data.yaml
```

### train / val / testの分割方針と比率

一般的な分割比率は **train:val:test = 7:2:1** ですが、データ数が少ない場合（200枚以下）は **8:1:1** にして学習データを確保するのが得策です。

- **train（訓練）：** モデルの重みを更新するために使う
- **val（検証）：** 学習中にmAPなどの指標をエポックごとに確認する
- **test（テスト）：** 学習終了後に一度だけ使う最終評価用（valと分けることで過楽観な評価を防ぐ）

### 分割スクリプト

```python
# scripts/split_dataset.py
import shutil
import random
from pathlib import Path

def split_dataset(
    images_dir: str,
    labels_dir: str,
    output_dir: str,
    ratios: tuple = (0.7, 0.2, 0.1),
    seed: int = 42,
):
    random.seed(seed)
    images = sorted(Path(images_dir).glob("*.jpg"))
    random.shuffle(images)

    n = len(images)
    n_train = int(n * ratios[0])
    n_val = int(n * ratios[1])

    splits = {
        "train": images[:n_train],
        "val": images[n_train : n_train + n_val],
        "test": images[n_train + n_val :],
    }

    for split, paths in splits.items():
        (Path(output_dir) / "images" / split).mkdir(parents=True, exist_ok=True)
        (Path(output_dir) / "labels" / split).mkdir(parents=True, exist_ok=True)
        for img_path in paths:
            label_path = Path(labels_dir) / (img_path.stem + ".txt")
            shutil.copy(img_path, Path(output_dir) / "images" / split / img_path.name)
            if label_path.exists():
                shutil.copy(label_path, Path(output_dir) / "labels" / split / label_path.name)

    for split, paths in splits.items():
        print(f"{split}: {len(paths)} 枚")

if __name__ == "__main__":
    split_dataset("annotations/images", "annotations/labels", "dataset/")
```

```bash
python scripts/split_dataset.py
# → train: 126 枚 / val: 36 枚 / test: 18 枚
```

### data.yaml の作成

```yaml
# dataset/data.yaml
path: ./dataset          # データセットのルートパス（絶対パスでも可）
train: images/train
val: images/val
test: images/test

nc: 1                    # クラス数
names:
  - screw
```

`nc` と `names` は必ず `classes.txt` の内容と一致させてください。

### データ数・クラスバランスの確認

学習前に各クラスの件数を把握しておくと、学習結果の解釈が楽になります。

```python
from pathlib import Path
from collections import Counter

labels_dir = Path("dataset/labels/train")
counter = Counter()
for txt in labels_dir.glob("*.txt"):
    for line in txt.read_text().strip().split("\n"):
        if line:
            class_id = int(line.split()[0])
            counter[class_id] += 1

print(counter)
# → Counter({0: 312, 1: 87})
```

クラス間の比率が 10:1 を超えるような極端なアンバランスがある場合は、少数クラスの画像を追加収集するか、データ拡張で対処します。

### Google DriveへのデータセットアップロードとColabへの転送

データセットが完成したら、Google Colabに渡すためにZIP化してGoogle Driveにアップロードします。

```bash
# ローカルターミナルで実行
cd project/
zip -r dataset.zip dataset/
```

作成した `dataset.zip` をGoogle DriveにGUIでアップロードするか、`google-drive-ocamlfuse` などでCLIからアップロードします。Colabで直接アップロードする場合は次章の冒頭で説明します。

## 4. YOLOで学習する（Google Colab・GPU）

### データセットの準備（Colab側）

2章でマウントしたGoogle Driveから `dataset.zip` を展開します。

```python
# Colabセル
import zipfile, os

DRIVE_PATH = "/content/drive/MyDrive/yolo_project"
ZIP_PATH   = f"{DRIVE_PATH}/dataset.zip"
WORK_DIR   = "/content/yolo_project"

os.makedirs(WORK_DIR, exist_ok=True)
with zipfile.ZipFile(ZIP_PATH, "r") as z:
    z.extractall(WORK_DIR)

print("展開完了:", os.listdir(WORK_DIR))
```

### モデルサイズの選び方

Ultralytics YOLOは用途に応じて複数のサイズが用意されています。

| モデル | パラメータ数 | 速度 | 精度 | 用途 |
|---|---|---|---|---|
| YOLOv8n（nano） | 3.2M | 最速 | 低め | エッジデバイス、試作 |
| YOLOv8s（small） | 11.2M | 速い | 中程度 | 一般的な用途 |
| YOLOv8m（medium） | 25.9M | 普通 | 高め | 精度重視 |
| YOLOv8l（large） | 43.7M | 遅い | 高い | 高精度が必要な場合 |

**まずは `yolov8s` で試し、精度が足りなければ `yolov8m` にアップグレード**するのがセオリーです。データ数が少ない段階でlargeモデルを選ぶと過学習しやすく、学習時間も無駄になります。

### 学習コードと主要パラメータの説明

```python
# Colabセル
import os
from ultralytics import YOLO

WORK_DIR   = "/content/yolo_project"
DRIVE_PATH = "/content/drive/MyDrive/yolo_project"

model = YOLO("yolov8s.pt")  # 事前学習済みの重みをダウンロードして使用

results = model.train(
    data=f"{WORK_DIR}/dataset/data.yaml",
    epochs=100,       # 学習エポック数。100〜200が一般的な出発点
    imgsz=640,        # 入力画像サイズ。640が標準（小さくすると速いが精度落ち）
    batch=16,         # T4の場合16が目安。A100なら32〜64まで増やせる
    patience=20,      # val mAPが改善しないエポックが続いたら早期終了
    device="cuda",    # Colab GPUを使用
    project=DRIVE_PATH,   # 結果をGoogle Driveに直接保存（セッション切れても消えない）
    name="exp1",
)
```

**主要パラメータの意味：**

- `epochs`：データセット全体を何回繰り返して学習するか。多すぎると過学習のリスク
- `patience`：早期終了（EarlyStopping）の猶予エポック数。過学習を自動で防ぐ
- `imgsz`：YOLOは入力前にこのサイズにリサイズする。元画像が大きい場合でも変換される
- `project`：Google Driveを指定することで、Colabセッションが切れても結果が保持される

### 学習の実行

上記のセルを実行します。学習が始まると、エポックごとに損失と指標が表示されます。

```
Epoch    GPU_mem   box_loss   cls_loss   dfl_loss  Instances       Size
  1/100     3.45G      2.341      3.012      1.456         24        640
  2/100     3.45G      1.987      2.543      1.312         18        640
  ...
```

T4 GPUでは1エポックあたり数十秒〜数分で完了します。100エポックの学習は通常30〜60分程度です。

学習が終わると `MyDrive/yolo_project/exp1/` に以下のファイルが生成されます。

```
MyDrive/yolo_project/exp1/
├── weights/
│   ├── best.pt      ← val mAPが最高だったときの重み
│   └── last.pt      ← 最終エポックの重み
├── results.csv      ← エポックごとの損失・mAP
└── results.png      ← 学習曲線の画像
```

### 学習済み best.pt の保存場所の確認

```python
# Colabセル
import os
weights_dir = f"{DRIVE_PATH}/exp1/weights"
print(os.listdir(weights_dir))
# → ['best.pt', 'last.pt']
```

評価には必ず `best.pt`（val mAP最大時点の重み）を使います。Google Driveに保存されているため、Colabセッションを再起動しても参照できます。

## 5. モデルの検証・評価（Google Colab・GPU）

### 定量評価（mAPの計算）

`model.val()` を使うと、検証データ全体に対する精度指標を一括で計算できます。

```python
# Colabセル
metrics = model.val(
    data=f"{WORK_DIR}/dataset/data.yaml",
    split="val",       # 評価対象を val セットに指定
    device="cuda",
    project=DRIVE_PATH,
    name="eval_val",
)

print(f"mAP50     : {metrics.box.map50:.3f}")
print(f"mAP50-95  : {metrics.box.map:.3f}")
print(f"Precision : {metrics.box.mp:.3f}")
print(f"Recall    : {metrics.box.mr:.3f}")
```

今回の出力結果：

![model.val()の出力結果](/images/yolo-video-labelstudio-training/image5.png)

### 評価指標の読み方

- **mAP50**：ボックスが「大体合っていれば正解」とした精度。**0.7以上で実用レベル**
- **mAP50-95**：ボックスの位置精度まで厳しく評価。0.5以上が目標
- **Precision**：誤検知の少なさ（「検出した」中で本当に正しかった割合）
- **Recall**：見逃しの少なさ（「実際に存在する」中で検出できた割合）

まず **mAP50** で実用性を判断し、問題なければ **mAP50-95** で位置精度を確認します。

## 6. 現状の精度と課題

本記事の手順で作ったモデルは、**学習データに近い条件（照明・角度・距離）では高い精度**が出る一方、以下のような状況で精度が落ちやすいです。

- 動画に映っていなかった新しい背景・照明条件
- 高速移動によるぼけ

このような場面での精度向上については、次回以降の記事で取り上げます。
