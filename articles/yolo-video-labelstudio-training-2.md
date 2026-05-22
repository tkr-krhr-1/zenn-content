---
title: "🧠動画データからYOLOモデルをゼロから作る——データセット準備・学習・評価編"
emoji: "⚙️"
type: "tech"
topics: ["yolo", "ultralytics", "python", "googlecolab", "物体検出"]
published: false
---

※本記事は、読者の方にとって分かりやすい構成となるようAIのサポートを活用し、私自身の経験と裏取りを元に書き下ろしたものです。

## はじめに

[前回の記事（フレーム抽出・アノテーション編）](https://zenn.dev/emp_tech_blog/articles/yolo-video-labelstudio-training)では、動画からフレームを抽出し、Label StudioでYOLO形式のアノテーションを行いました。

本記事ではその続きとして、アノテーション済みデータをYOLO学習用データセットに整形し、Google ColabのGPUで学習・評価するまでの工程を解説します。

![挿絵](/images/yolo-video-labelstudio-training-2/image0.png)

### 使用技術スタック

| 役割 | ツール / ライブラリ |
|---|---|
| 物体検出フレームワーク | Ultralytics YOLOv8 |
| 学習・評価環境 | Google Colab（GPU） |
| データセット準備 | Python（shutil / pathlib） |

### 作業環境の全体像

| 章 | 作業内容 | 実行環境 |
|---|---|---|
| 1章 | Google Colab環境構築 | **Google Colab** |
| 2章 | データセット準備・アップロード | **ローカル** |
| 3章 | YOLOで学習 | **Google Colab（GPU）** |
| 4章 | モデルの評価 | **Google Colab（GPU）** |

### 記事の流れ

```
【ローカルPC】
  ↓ 2章：train/val/test分割 + data.yaml
YOLOデータセット（ZIP）→ Google Driveにアップロード

【Google Colab（GPU）】
  ↓ 3章：学習（NVIDIA T4 / A100）
best.pt（学習済みモデル）→ Google Driveに保存
  ↓ 4章：評価
評価指標（mAP / Precision / Recall）
```

## 1. Google Colab環境構築

3章の学習はGoogle Colabで実行します。新しいノートブックを開き、以下の手順でセットアップします。

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

## 2. データセットの準備（ローカル）

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

アノテーションしたファイルを**train:val:test = 7:2:1** で分割します。
この比率は広く使われている**慣習**であり、理論的に最適と証明された値ではありません。
データ量・クラス数・クラスの偏りによって適切な比率は変わるため、少数クラスが val や test に十分含まれるかどうかを確認してください。

- **train（訓練）：** モデルの重みを更新するために使う
- **val（検証）：** 学習中にmAPなどの指標をエポックごとに確認する
- **test（テスト）：** 学習終了後に一度だけ使う最終評価用（valと分けることで過楽観な評価を防ぐ）

### 前回記事からの引き継ぎファイル

分割スクリプトは、[前回の記事](https://zenn.dev/emp_tech_blog/articles/yolo-video-labelstudio-training)でLabel StudioからエクスポートしたデータをそのままYOLO形式で入力として受け取ります。エクスポート後のディレクトリ構成は以下を想定しています。

```
annotations/
├── images/    ← Label Studioからエクスポートした画像ファイル群
└── labels/    ← 各画像に対応するYOLO形式のラベルファイル（.txt）
```

`labels/` 内の各 `.txt` は、対応する画像と同じファイル名で `クラスindex cx cy w h` の形式で記録されています（例：`frame_0001.txt`）。

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

`data.yaml` は、YOLOに「どこにデータがあるか」「何クラスを検出するか」を伝える設定ファイルです。
学習・評価コマンドはすべてこのファイルを参照して動くため、パスやクラス定義が間違っていると学習自体が失敗します。

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
`classes.txt` は[前回の記事](https://zenn.dev/emp_tech_blog/articles/yolo-video-labelstudio-training)でLabel Studioからエクスポートした際に生成されたファイルで、各行がクラス名に対応しています（例：1行目が `screw` なら index=0）。`names` リストの並び順がこのファイルと異なると、学習後の推論でクラスラベルがずれます。

### Google DriveへのデータセットアップロードとColabへの転送

データセットが完成したら、Google Colabに渡すためにZIP化してGoogle Driveにアップロードします。

```bash
# ローカルターミナルで実行
cd project/
zip -r dataset.zip dataset/
```

作成した `dataset.zip` をGoogle DriveにGUIでアップロードするか、`google-drive-ocamlfuse` などでCLIからアップロードします。Colabで直接アップロードする場合は次章の冒頭で説明します。

## 3. YOLOで学習する（Google Colab・GPU）

### データセットの準備（Colab側）

1章でマウントしたGoogle Driveから `dataset.zip` を展開します。

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

### 学習コードと主要パラメータの説明

```python
# Colabセル
import os
from ultralytics import YOLO

WORK_DIR   = "/content/yolo_project"
DRIVE_PATH = "/content/drive/MyDrive/yolo_project"

model = YOLO("yolov8s.pt")  # データが少ない初回はnano/smallから始める

results = model.train(
    data=f"{WORK_DIR}/dataset/data.yaml",
    epochs=100,
    imgsz=640,
    batch=16,
    patience=20,
    device="cuda",
    project=DRIVE_PATH,
    name="exp1",
)
```

**パラメータの意味：**

- `data`：学習・検証データのパスやクラス定義を記述した `data.yaml` のパス。すべての工程はこのファイルを起点にする
- `epochs`：データセット全体を何回繰り返して学習するか。多すぎると過学習のリスクがある。100〜200が一般的な出発点
- `imgsz`：入力画像の短辺をこのサイズにリサイズして学習する。640が標準。小さくすると速度は上がるが小さな物体の検出精度が落ちる
- `batch`：1回の重み更新に使う画像枚数。T4では16が目安。大きくすると学習が安定するがGPUメモリを消費する
- `patience`：val mAP が改善しないエポックが続いたら学習を自動で打ち切る猶予エポック数（EarlyStopping）。過学習を防ぐ
- `device`：学習に使うデバイス。`"cuda"` でGPU、`"cpu"` でCPU。Colabでは `"cuda"` を指定する
- `project`：結果の保存先ディレクトリ。Google Driveを指定することでColabセッションが切れても結果が消えない
- `name`：`project` 以下に作られるサブディレクトリ名。実験ごとに変えると結果が上書きされない

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

評価には必ず `best.pt`（val mAP最大時点の重み）を使います。

Google Driveに保存されているため、Colabセッションを再起動しても参照できます。

## 4. モデルの検証・評価（Google Colab・GPU）

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

![model.val()の出力結果](/images/yolo-video-labelstudio-training-2/image1.png)

### 評価指標の読み方

- **mAP50**：ボックスが「大体合っていれば正解」とした精度。**0.7以上で実用レベル**
- **mAP50-95**：ボックスの位置精度まで厳しく評価。0.5以上が目標
- **Precision**：誤検知の少なさ（「検出した」中で本当に正しかった割合）
- **Recall**：見逃しの少なさ（「実際に存在する」中で検出できた割合）

まず **mAP50** で実用性を判断し、問題なければ **mAP50-95** で位置精度を確認します。

## 5. まとめ

本記事では、Label Studioでアノテーションしたデータを起点に、YOLO学習用データセットの整形・分割から、Google ColabでのGPU学習・評価までを一通り解説しました。

| 工程 | 要点 |
|---|---|
| データセット分割 | train:val:test = 7:2:1 は慣習。少数クラスが val/test に十分入るか確認する |
| `data.yaml` | `nc` と `names` の順番を `classes.txt` と必ず一致させる |
| 学習 | `patience` で過学習を自動防止。モデルはGoogle Driveに保存してセッション切れに備える |
| 評価 | まず `mAP50 ≥ 0.7` を目標に。`best.pt` を使って最終評価する |
