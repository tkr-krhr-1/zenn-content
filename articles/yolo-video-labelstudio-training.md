---
title: "🎬動画データからYOLOモデルをゼロから作る——フレーム抽出とLabel Studioアノテーション編"
emoji: "🎬"
type: "tech"
topics: ["yolo", "ultralytics", "python", "labelstudio", "物体検出"]
published: false
---

※本記事は、読者の方にとって分かりやすい構成となるようAIのサポートを活用し、私自身の経験と裏取りを元に書き下ろしたものです。

## 1. はじめに

### 作るもの

この記事では、手持ちの動画ファイルから独自のYOLO物体検出モデルを構築するための**前半工程**——フレーム抽出とLabel Studioを使ったアノテーション——を解説します。

動画 → フレーム抽出 → アノテーション → 学習 → 検証という一方通行の流れを体験することで、「公開データセットがない対象」でも自前モデルが作れるようになります。

![](/images/yolo-video-labelstudio-training/image0.png =750x)

データセット学習・評価は後続記事（🧠動画データからYOLOモデルをゼロから作る——データセット準備・学習・評価編）で解説します。

### 対象読者・前提知識

- Pythonの基礎文法が読める方
- YOLOの概念（バウンディングボックス、クラス、mAPなど）をざっくり知っている方
- 物体検出モデルを自作したいが、どこから始めればいいか分からない方

YOLOの基礎から学びたい方は、まず[入門編](https://zenn.dev/tkr_krhr/articles/yolo-object-detection-intro)をご覧ください。

### 使用技術スタック

| 役割 | ツール / ライブラリ |
|---|---|
| 物体検出フレームワーク | Ultralytics YOLOv8 |
| アノテーション | Label Studio |
| 動画処理 | OpenCV（cv2） |

### 作業環境の全体像

この記事ではすべて**ローカルPC**で作業します。

| 章 | 作業内容 | 実行環境 |
|---|---|---|
| 2章 | 環境構築 | **ローカル** |
| 3章 | フレーム抽出 | **ローカル** |
| 4章 | Label Studioでアノテーション | **ローカル** |

### 記事の流れ

```
【ローカルPC】
動画ファイル
  ↓ 3章：フレーム抽出・間引き
静止画（PNG/JPG）× 数百枚
  ↓ 4章：Label Studioでアノテーション
YOLO形式ラベルファイル → 後続記事へ続く
```

## 2. 環境構築

#### 仮想環境の作成

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
```

#### Pythonライブラリのインストール

```bash
pip install opencv-python-headless scikit-image label-studio
```

#### Label StudioのセットアップとURL確認

インストール後、以下のコマンドで起動します。

```bash
label-studio start
```

ブラウザで `http://localhost:8080` にアクセスし、アカウントを作成したらセットアップ完了です。

## 3. 動画から画像データを作成する

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

## 4. Label Studioでアノテーション

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
2. 左下のラベルパネルからクラスを選択
3. 画像上をドラッグしてバウンディングボックスを描画
4. 右下に「Submit」が出てくるので押下でラベルを確定

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

アノテーションが完成したら、次はデータセットの準備と学習です。

[後続記事（学習・評価編）](https://zenn.dev/tkr_krhr/articles/yolo-video-labelstudio-training-2)に続きます。
