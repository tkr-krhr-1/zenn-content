---
title: "動画データからYOLOモデルをゼロから作る——Label Studioでアノテーションして独自物体検出器を構築する"
emoji: "🎬"
type: "tech"
topics: ["yolo", "ultralytics", "python", "labelstudio", "物体検出"]
published: false
---

※本記事は、読者の方にとって分かりやすい構成となるようAIのサポートを活用し、私自身の経験と裏取りを元に書き下ろしたものです。

## 1. はじめに

### 作るもの

この記事では、手持ちの動画ファイルから、独自のYOLO物体検出モデルをイチから構築する方法を解説します。

完成したモデルに動画を入力すると、対象の物体をリアルタイムで検出してバウンディングボックスを描画できます。

![完成イメージ：動画への推論結果](/images/yolo-video-labelstudio-training/hero.png)

動画 → フレーム抽出 → アノテーション → 学習 → 検証という一方通行の流れを体験することで、「公開データセットがない対象」でも自前モデルが作れるようになります。

### 対象読者・前提知識

- Pythonの基礎文法が読める方
- YOLOの概念（バウンディングボックス、クラス、mAPなど）をざっくり知っている方
- 物体検出モデルを自作したいが、どこから始めればいいか分からない方

YOLOの基礎から学びたい方は、まず[入門編](https://zenn.dev/tkr_krhr/articles/yolo-object-detection-intro)をご覧ください。

### 使用技術スタック

| 役割 | ツール / ライブラリ |
|---|---|
| 物体検出フレームワーク | Ultralytics YOLO（YOLOv8〜v11） |
| アノテーション | Label Studio |
| 動画処理 | OpenCV（cv2） |
| 類似フレーム除去 | scikit-image / NumPy |
| データ管理 | Python標準ライブラリ（shutil, pathlib） |

### 作業環境の全体像

この記事では、**ローカルPC**と**Google Colab**の2つの環境を使い分けます。

| 章 | 作業内容 | 実行環境 |
|---|---|---|
| 2章 | 環境構築 | ローカル / Colab |
| 3章 | フレーム抽出・間引き | **ローカル** |
| 4章 | Label Studioでアノテーション | **ローカル** |
| 5章 | データセット準備・アップロード | **ローカル** |
| 6章 | YOLOで学習 | **Google Colab（GPU）** |
| 7章 | モデルの検証・評価 | **Google Colab（GPU）** |

ローカルで手間のかかるアノテーション作業を行い、学習・評価はColabの無料GPUに任せる構成です。

### 記事の流れ

```
【ローカルPC】
動画ファイル
  ↓ 3章：フレーム抽出・間引き
静止画（PNG/JPG）× 数百枚
  ↓ 4章：Label Studioでアノテーション
YOLO形式ラベルファイル
  ↓ 5章：train/val/test分割 + data.yaml
YOLOデータセット（ZIP）→ Google Driveにアップロード

【Google Colab（GPU）】
  ↓ 6章：学習（NVIDIA T4 / A100）
best.pt（学習済みモデル）→ Google Driveに保存
  ↓ 7章：評価・推論
検出結果動画・評価指標
```

---

## 2. 環境構築

### ローカルPC環境

#### 動作確認済み環境

| 項目 | バージョン |
|---|---|
| OS | macOS 15 / Ubuntu 22.04 |
| Python | 3.10 以上 |
| Label Studio | 1.13.x |

フレーム抽出・アノテーション・データセット整理はローカルで行います。学習はColabに任せるため、**ローカルにGPUは不要**です。

#### 仮想環境の作成

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
```

#### Pythonライブラリのインストール（ローカル用）

```bash
pip install opencv-python-headless scikit-image label-studio
```

#### Label StudioのセットアップとURL確認

インストール後、以下のコマンドで起動します。

```bash
label-studio start
```

ブラウザで `http://localhost:8080` にアクセスし、アカウントを作成したらセットアップ完了です。

---

### Google Colab環境（GPU）

6章の学習はGoogle Colabで実行します。新しいノートブックを開き、以下の手順でセットアップします。

#### GPUランタイムの有効化

「ランタイム」→「ランタイムのタイプを変更」ランタイムを **T4 GPU** に設定します。

#### GPU接続の確認

```python
# Colabセル
import torch
print(torch.cuda.is_available())   # True であればGPU有効
print(torch.cuda.get_device_name(0))  # 例: Tesla T4
```

#### ライブラリのインストール（Colab用）

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

### ディレクトリ構成の全体像

作業を進める前に、最終的なディレクトリ構成を把握しておくと迷いが減ります。

```
project/
├── videos/              # 元動画を置く
├── frames/              # 抽出した静止画
├── annotations/         # Label Studioからエクスポートしたラベル
├── dataset/
│   ├── images/
│   │   ├── train/
│   │   ├── val/
│   │   └── test/
│   ├── labels/
│   │   ├── train/
│   │   ├── val/
│   │   └── test/
│   └── data.yaml
├── runs/                # 学習結果（YOLOが自動生成）
└── scripts/
    ├── extract_frames.py
    ├── split_dataset.py
    └── train.py
```

---

## 3. 動画から画像データを作成する（ローカル）

### フレーム抽出の考え方

動画からすべてのフレームを取り出せば学習データは増えますが、**それが精度向上につながるとは限りません**。

- **多すぎる場合：** ほぼ同じ構図の画像が数百枚並び、モデルが特定の視点に過適合する。アノテーション工数も膨大になる。
- **少なすぎる場合：** 検出したいシーンをカバーできず、汎化性能が落ちる。

目安として「同じシーンが連続する場合は1〜2秒に1枚、動きが速い場合は0.5秒に1枚」程度からスタートするのが現実的です。

### 抽出間隔の決め方

動画のフレームレート（fps）に基づいて、何フレームおきに1枚取り出すかを計算します。

```
interval = fps × 秒数
例）30fps の動画から1秒おきに抽出したい → interval = 30
```

一般的な防犯カメラ映像（30fps）で、1秒おきに抽出すると、1分の動画から60枚が得られます。

### フレーム抽出コード

```python
# scripts/extract_frames.py
import cv2
from pathlib import Path

def extract_frames(video_path: str, output_dir: str, interval: int = 30):
    """動画からintervalフレームおきに静止画を保存する。"""
    output_dir = Path(output_dir)
    output_dir.mkdir(parents=True, exist_ok=True)

    cap = cv2.VideoCapture(video_path)
    fps = cap.get(cv2.CAP_PROP_FPS)
    total = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
    print(f"fps={fps:.1f}, 総フレーム数={total}, 抽出間隔={interval}フレーム")

    frame_idx = 0
    saved = 0
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break
        if frame_idx % interval == 0:
            filename = output_dir / f"frame_{frame_idx:06d}.jpg"
            cv2.imwrite(str(filename), frame)
            saved += 1
        frame_idx += 1

    cap.release()
    print(f"保存枚数: {saved} 枚 → {output_dir}")

if __name__ == "__main__":
    extract_frames("videos/input.mp4", "frames/", interval=30)
```

### 抽出結果の確認

```bash
python scripts/extract_frames.py
# → 保存枚数: 180 枚 → frames/
```

画像を数枚開いて、ブレやピンぼけが多くないか確認します。暗すぎる・明るすぎるフレームは学習ノイズになるため、目視で除外しておきます。

---

## 4. Label Studioでアノテーション（ローカル）

### プロジェクト作成とラベルの設定

Label Studioにログインし、「Create Project」でプロジェクトを作成します。

1. 「Object Detection with Bounding Boxes」テンプレートを選択

![テンプレート選択画面](/images/yolo-video-labelstudio-training/image1.png)

2. 「Add Label」でクラス名を追加（今回は `screw`）
3. 「Save」で確定

![screwラベルを追加したLabeling Interface設定画面](/images/yolo-video-labelstudio-training/image2.png)

### 抽出画像のインポート（ローカルストレージ経由）

:::message alert
直接インポート（ドラッグ&ドロップ）は**使わないでください**。直接インポートするとファイル名とパスが内部で別物に変換され、アノテーション後にラベルと元画像の対応が取れなくなります。必ずローカルストレージ設定を通じてインポートしてください。
:::

#### 1. 環境変数を設定してLabel Studioを起動する

Label Studioがローカルファイルを配信できるよう、以下の環境変数を設定してから起動します。

```bash
export LABEL_STUDIO_LOCAL_FILES_SERVING_ENABLED=true
export LABEL_STUDIO_LOCAL_FILES_DOCUMENT_ROOT=/path/to/project   # framesの親ディレクトリを指定

label-studio start
```

`LABEL_STUDIO_LOCAL_FILES_DOCUMENT_ROOT` には `frames/` が含まれる親ディレクトリを指定します。プロジェクトルートが `/Users/yourname/project` であれば、そのパスをそのまま設定します。

#### 2. ローカルストレージをプロジェクトに追加する

Label StudioのUIで以下の手順を実行します。

1. 対象プロジェクトの **Settings > Cloud Storage** に移動
2. **Add Source Storage** をクリック
3. **Storage Type** で「**Local files**」を選択
4. **Local Path** に `frames/` ディレクトリのフルパスを入力（例：`/Users/yourname/project/frames`）
5. **「Treat every bucket object as a source file」** にチェックを入れる
6. **Check Connection** でアクセスできることを確認
7. **Add Storage** で保存
8. **Sync Storage** をクリックして画像を同期

同期が完了すると、タスク一覧に `frames/` 内の画像が並びます。ファイルパスが保持されているため、エクスポート後もラベルと元画像を正しく紐付けられます。

![同期完了後のタスク一覧（121枚インポート済み）](/images/yolo-video-labelstudio-training/image3.png)

### アノテーション作業の手順

1. タスク一覧から画像を選択してアノテーション画面を開く
2. 左のラベルパネルからクラスを選択（キーボードショートカット `1`、`2`…で切り替え可能）
3. 画像上をドラッグしてバウンディングボックスを描画
4. 右上の「Submit」でラベルを確定

![アノテーション作業画面（ねじにバウンディングボックスを描画）](/images/yolo-video-labelstudio-training/image4.png)

**バウンディングボックスのコツ：**

- 対象物体の輪郭ぴったりに合わせる（余白を入れすぎない）
- 一部が隠れている（遮蔽）場合も、見えている範囲だけをラベル
- 小さすぎて判別できない物体はラベルしない（32×32px未満が目安）

### 効率化のコツ

| 操作 | ショートカット |
|---|---|
| ラベル切り替え | `1`, `2`, `3`... |
| ボックス移動 | 矢印キー |
| ボックス削除 | `Delete` |
| 次のタスク | `D` |
| 前のタスク | `A` |

複数の画像で同じ位置に物体が映っている場合、最初の画像でラベルを付けた後「Copy Annotation」を使うと次の画像に引き継げます。

### YOLO形式でエクスポート

全画像のアノテーションが終わったら、エクスポートします。

1. プロジェクトのトップページ「Export」をクリック
2. フォーマットで「YOLO with Images」を選択
3. ダウンロードされたZIPを解凍して `annotations/` に配置

### エクスポートデータの中身の確認

解凍すると以下の構成になっています。

```
annotations/
├── images/         # 元画像（JPG）
├── labels/         # バウンディングボックスのテキスト
│   └── frame_000030.txt
└── classes.txt     # クラス名のリスト
```

`labels/` 内のテキストファイルは1行1物体で、YOLO形式（クラスID + 正規化座標）になっています。

```
0 0.512 0.334 0.186 0.402
```

`class_id x_center y_center width height`（すべて0〜1の相対値）

---

## 5. データセットの準備（ローカル）

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

クラス間の比率が 10:1 を超えるような極端なアンバランスがある場合は、少数クラスの画像を追加収集するか、後述のデータ拡張で対処します。

### Google DriveへのデータセットアップロードとColabへの転送

データセットが完成したら、Google Colabに渡すためにZIP化してGoogle Driveにアップロードします。

```bash
# ローカルターミナルで実行
cd project/
zip -r dataset.zip dataset/
```

作成した `dataset.zip` をGoogle DriveにGUIでアップロードするか、`google-drive-ocamlfuse` などでCLIからアップロードします。Colabで直接アップロードする場合は次章の冒頭で説明します。

---

## 6. YOLOで学習する（Google Colab・GPU）

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

ローカルから直接アップロードする場合は、以下でColabの一時ストレージにアップロードできます（セッション終了で消えるため注意）。

```python
# Colabセル（Driveを使わない場合）
from google.colab import files
uploaded = files.upload()   # ダイアログが開くのでdataset.zipを選択
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

### 学習曲線（loss・mAP）の確認

学習が終わると `MyDrive/yolo_project/exp1/` に以下のファイルが生成されます。

```
MyDrive/yolo_project/exp1/
├── weights/
│   ├── best.pt      ← val mAPが最高だったときの重み
│   └── last.pt      ← 最終エポックの重み
├── results.csv      ← エポックごとの損失・mAP
└── results.png      ← 学習曲線の画像
```

`results.png` を開き、以下を確認します。

- **box_loss / cls_loss / dfl_loss** が単調に下がっているか（発散していないか）
- **val/mAP50** がエポックを重ねて上昇しているか
- trainとvalの損失が極端に乖離していないか（過学習のサイン）

### 学習済み best.pt の保存場所の確認

```python
# Colabセル
import os
weights_dir = f"{DRIVE_PATH}/exp1/weights"
print(os.listdir(weights_dir))
# → ['best.pt', 'last.pt']
```

推論・評価には必ず `best.pt`（val mAP最大時点の重み）を使います。Google Driveに保存されているため、Colabセッションを再起動しても参照できます。

---

## 8. 現状の精度と課題

本記事の手順で作ったモデルは、**学習データに近い条件（照明・角度・距離）では高い精度**が出る一方、以下のような状況で精度が落ちやすいです。

- 動画に映っていなかった新しい背景・照明条件
- 高速移動によるモーションブラー

このような場面での精度向上については、次回の記事で取り上げます。
