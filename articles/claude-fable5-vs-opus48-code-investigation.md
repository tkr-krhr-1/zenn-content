---
title: "Claude Fable 5 vs Opus 4.8 をClaude Codeで実測した——改修計画タスクの比較"
emoji: "🔍"
type: "tech"
topics: ["claudecode", "claude", "anthropic", "ai", "llm"]
published: false
---

※本記事は、読者の方にとって分かりやすい構成となるようAIのサポートを活用し、私自身の経験と裏取りを元に書き下ろしたものです。

## はじめに

2026年6月10日(日本時間)、Anthropicが新しい最上位モデル **Claude Fable 5** をリリースしました。これまでのOpus・Sonnet・Haikuの3階層の上に新設された「Mythos級」と呼ばれる性能帯のモデルで、一般提供されるモデルとしては現時点で最高性能とされています。

公式ベンチマークの数字はかなり強烈です。

| ベンチマーク | Fable 5 | Opus 4.8 |
|---|---|---|
| [SWE-Bench Pro](https://labs.scale.com/leaderboard/swe_bench_pro_public)(エージェント型コーディング) | 80.3% | 69.2% |
| [FrontierCode Diamond](https://cognition.ai/blog/frontier-code)(マージ可能な品質か) | 29.3% | 13.4% |

特にFrontierCode Diamondは「動くか」ではなく「人間のメンテナーが実際にマージできる品質か」を測る指標で、ここで2倍以上の差がついています。

……とはいえ、ベンチマークの数字を眺めていても体感はわかりません。日々Claude Codeを使う側として知りたいのは、もっと泥臭い話です。

- **同じタスクを投げたとき、どちらが速く終わるのか**
- **どちらが無駄なくファイルを読み、トークンを節約してくれるのか**

そこで本記事では、Claude Code上でFable 5とOpus 4.8に**完全に同一のタスク**を実行させ、所要時間・トークン消費・ツールコール数を実測して比較しました。

今回は「**改修計画タスク**」——既存プロジェクトの改修計画書を作成する（コードは変更しない）——で両モデルを比較します。

:::message
対象プランでは**2026年6月22日まで**追加費用なしでFable 5を試せる期間が設けられています(執筆時点)。気になる方は早めに触ってみるのがおすすめです。条件は変わる可能性があるので公式の最新情報を確認してください。
:::

## Fable 5が生まれた背景

Claude Fable 5は、Anthropicが「フロンティアモデル競争の次フェーズ」として投入したモデルです。Opusが「高性能だが重い」、Sonnetが「バランス型」という位置づけだったのに対して、Fable 5は**速度を犠牲にせず知能を引き上げる**ことを設計目標に掲げています。

SWE-Bench Proでのスコア向上(69.2% → 80.3%)が示すのは、単純なコード生成の精度だけでなく、**問題を自律的に分解して解決するエージェント能力の向上**です。コーディングエージェントとして使うとき、この差は「何度もやり直させるか、一発で終わるか」に直結します。

## 準備

### 必要なもの

- Claude Codeがインストール済みであること
- Fable 5を利用できるプランに加入していること(2026年6月22日まで追加費用なしで試せる期間あり、執筆時点)
- 今回の検証対象リポジトリ: [fastapi/full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template)

### モデルの切り替え方

Claude Code上でのモデル切り替えは `/model` コマンドで行います。

```
/model claude-fable-5    # Fable 5に切り替え
/model claude-opus-4-8   # Opus 4.8に切り替え
```

### コスト取得の方法

実行後に `/cost` を入力すると、そのセッションのトークン消費の内訳が表示されます。input・output・cache readの3項目が出るので、実行直後に必ずメモしてください。

### 公平な比較のためのルール

LLMの出力にはブレがあるので、条件を揃えたうえで複数回実行しました。

- プロンプトは**完全に同一の文面**をコピーして使用
- セッションは毎回新規に起動(`claude`を起動し直す)
- 同一コミット状態のリポジトリから開始
- CLAUDE.mdなどの追加コンテキストは一切置かない(素の状態で比較)
- **各モデル2回ずつ実行**し、両方の値を記録

full-stack-fastapi-template を選んだのは、FastAPI + SQLAlchemy のバックエンドと React フロントエンドを持つ中規模プロジェクトで、CRUD 操作が直接 SQLAlchemy Session で書かれており、「Repository パターンへの移行計画」というタスクが自然に成立するためです。

### 検証環境

- Claude Code v2.1.172
- 対象リポジトリ: [fastapi/full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template)(commit `38302d7`)
- マシン: MacBook Air M4チップ

## 検証: 改修計画タスク

### タスク内容

対象リポジトリは [fastapi/full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template) です。FastAPI + SQLAlchemy のバックエンドと React フロントエンドを持つ中規模プロジェクトで、CRUD 操作が直接 SQLAlchemy Session で書かれています。

```bash
git clone --depth 1 https://github.com/fastapi/full-stack-fastapi-template /tmp/full-stack-fastapi-template
```

両モデルに投げたプロンプトはこれだけです。

```
このプロジェクトのバックエンドを調査して、CRUDの実装を
現在の直接的なSQLAlchemy Session操作から Repository パターンへ
移行するための実装計画を作成してください。

計画には以下を含めてください：
- 変更が必要なファイルの一覧と、各ファイルで何を変えるか
- 作業の推奨順序と、その理由
- 移行中に既存のテストを壊さないための方針
- 見落としやすいリスクや注意点

コードは書かないでください。計画書のみ作成してください。
```

コードを書かせない点が検証1との最大の違いです。「どこを変えるべきか」の把握力と、計画書としての完成度が直接比べられます。

### 計測値サマリー

| 計測項目 | Fable 5 (1回目) | Fable 5 (2回目) | Opus 4.8 (1回目) | Opus 4.8 (2回目) |
|---|---|---|---|---|
| 所要時間 | 3分37秒 | 3分54秒 | 3分57秒 | 3分28秒 |
| inputトークン | 3,624 | 3,624 | 3,750 | 3,750 |
| outputトークン | 11,261 | 12,159 | 14,674 | 13,206 |
| cache read | 213,697 | 227,044 | 308,805 | 323,030 |
| ツールコール総数 | 15回 | 15回 | 18回 | 15回 |
| (内訳)Bash | 2 | 2 | 4 | 4 |
| (内訳)Read | 12 | 12 | 13 | 10 |
| コスト(USD) | $1.68 | $1.48 | $1.09 | $0.87 |

### 生成された計画書

:::details Fable 5 が作成した計画書（run1 の要約）
Fable 5 は計画書をファイルに書き込んだあと、セッション内で以下の要点サマリーを出力しました。

---

計画書を `backend/docs/repository-pattern-migration-plan.md` に作成しました。

**現状の把握**

CRUD 実装は二重構造になっています。`app/crud.py` のモジュール関数（user 4 つ + item 1 つ）に加え、`items.py` は全 5 エンドポイント、`users.py` は 7 箇所がルートハンドラ内で直接 `session.add/commit/get` を呼んでいます。さらに `deps.py`（`get_current_user`）、`core/db.py`（`init_db`）、`private.py` にも直接操作があり、テスト側も 5 ファイルが `crud` を直接 import しています。

**計画の骨子**

目標設計: `app/repositories/` に Session をコンストラクタ注入する `UserRepository` / `ItemRepository` を作り、`deps.py` の DI で注入。`authenticate` だけは Repository に入れず、セキュリティロジック（タイミング攻撃対策・ハッシュ自動アップグレード）として別の場所に移します。

**作業順序（6 フェーズ）:**

1. Repository 層を純粋追加（既存コード無変更）+ 単体テスト追加
2. `crud.py` を Repository への**委譲シム**に書き換え — 既存テスト全部が新実装を検証してくれるのがこのステップの肝
3. ルートをリスク昇順に 1 ファイルずつ移行: `items.py` → `private.py` → `users.py` → `login.py`（authenticate を含むので最後）
4. `deps.py` と `core/db.py`（conftest の起点なので単独コミット）
5. テストヘルパーと crud テストの移行
6. `crud.py` 削除

**テストを壊さない方針:** シムで `crud.py` の公開 API を最後まで不変に保つこと、そして「メソッド内 commit + refresh」のセマンティクスを維持することが最重要です。

**特に見落としやすいリスク（計画書には 10 項目記載）:**
- `authenticate` の DUMMY_HASH によるタイミング攻撃対策はテストが存在しないため、移設時に黙って消えやすい
- `delete_user` の Item 一括 DELETE はモデルの cascade 設定と重複しているが、移行ついでに「整理」しない
- `read_user_by_id` は 403 が 404 より先に返るチェック順（存在秘匿）で、素直に書き直すと挙動が変わる
- Repository を `SessionDep` 以外から生成すると、`current_user`（依存関係がロード済み）と別 Session になり、アタッチ済みインスタンス問題が起きる

---
:::

:::details Opus 4.8 が作成した計画書（run1 の全文）

# Repository パターン移行計画書

## 1. 現状分析

### CRUD 実装の現在地

CRUD ロジックは 2 か所に分散している。

1. **`app/crud.py`** — 関数ベースの CRUD（5 関数）
   - `create_user` / `update_user` / `get_user_by_email` / `authenticate` / `create_item`
   - すべて `session` を引数で受け取り、関数内部で `commit` まで行う
2. **ルートハンドラ内の直接 Session 操作** — `crud.py` を経由せず `session.get` / `session.exec` / `session.add` / `session.commit` を直接呼んでいる箇所が多数（特に `items.py` と `users.py`）

### 直接 Session 操作の所在（アプリ側）

| ファイル | 直接操作の内容 |
|---|---|
| `app/api/routes/users.py` | `read_users`（count + select）、`update_user_me`（`sqlmodel_update` + commit）、`update_password_me`、`delete_user_me`、`read_user_by_id`（`session.get`）、`delete_user`（Item の bulk delete + user delete を **1 トランザクション**で実行） |
| `app/api/routes/items.py` | 全 5 エンドポイントが直接操作 |
| `app/api/routes/private.py` | `create_user` が `User` を直接構築して add/commit（**refresh なし**、LOCAL 環境のみ有効） |
| `app/api/deps.py` | `get_current_user` 内の `session.get(User, ...)` |
| `app/core/db.py` | `init_db` 内の superuser 検索 select + `crud.create_user` 呼び出し |

## 2. 設計方針

- `app/repositories/` に `UserRepository` / `ItemRepository` を新設。コンストラクタで `Session` を受け取る
- ジェネリックな `BaseRepository[T]` はこの規模では**作らない**（モデル 2 つでは共通化よりメール検索・owner 絞り込みの差の方が大きい）
- commit 境界は第一段階では現挙動維持（Repository 内 commit）、Unit of Work 化は別タスク

**Repository に入れないもの:** `authenticate`（パスワード検証・タイミング攻撃対策・ハッシュアップグレードはドメインロジックであり永続化の責務ではない）

## 3. 変更ファイル一覧

### 新規作成

| ファイル | 内容 |
|---|---|
| `app/repositories/__init__.py` | パッケージ化、re-export |
| `app/repositories/user.py` | `UserRepository` |
| `app/repositories/item.py` | `ItemRepository` |
| `tests/repositories/test_user_repository.py` | `tests/crud/test_user.py` の移植版 |
| `tests/repositories/test_item_repository.py` | Item 用テスト（新規） |

### 変更

| ファイル | 変更内容 |
|---|---|
| `app/crud.py` | Phase 1 で全関数を Repository への委譲に書き換え、Phase 6 で削除 |
| `app/api/deps.py` | Repository の DI プロバイダ追加。`get_current_user` の `session.get` 置換 |
| `app/api/routes/items.py` | 全エンドポイントを `ItemRepoDep` 経由に置換 |
| `app/api/routes/private.py` | 直接操作を `UserRepository` 経由に置換 |
| `app/api/routes/login.py` | `crud.authenticate` の呼び出し先を新配置に置換 |
| `app/api/routes/users.py` | count/select・`session.get`・bulk delete をすべて Repository 経由に置換 |
| `app/core/db.py` | `init_db` の `crud.create_user` を `UserRepository` に置換 |
| `tests/utils/user.py` | `crud.*` → `UserRepository(db)` 経由に |
| `tests/utils/item.py` | 同上（`ItemRepository`） |

### 変更不要（明示）

- `tests/api/routes/test_*.py` の `db` fixture 直接操作（テストの arrange/assert は生 Session で問題ない）
- `tests/conftest.py`、`app/models.py`、`alembic/`

## 4. 推奨作業順序

ストラングラー方式（追加 → 委譲 → 置換 → 削除）を取る。**各 Phase 完了時点で全テストが green であること**を不変条件とする。

| Phase | 内容 | 理由 |
|---|---|---|
| 1 | Repository 新設 + `crud.py` を委譲ラッパー化 | 既存テスト全部が新実装の回帰テストとして機能する |
| 2 | Repository のテスト追加 | 以降の Phase で Repository 側に手を入れるとき直接検証できる |
| 3 | `deps.py` に DI 追加 | ルート移行の前提。全 API テストで配線ミスを最速検出 |
| 4 | ルートを 1 ファイルずつ移行（`items.py` → `private.py` → `login.py` → `users.py`） | 単純→複雑の順。罠が最も多い `users.py` を最後に |
| 5 | `core/db.py` とテストユーティリティ | フィクスチャ基盤なので本体の移行が安定してから |
| 6 | `app/crud.py` 削除（参照ゼロを確認してから） | — |

## 5. 既存テストを壊さないための方針

1. `crud.py` を即削除しない — Phase 1〜5 の間、委譲ラッパーとして残す
2. API テスト（`tests/api/routes/`）を回帰テストの主軸にする
3. 旧テストと新テストの並走期間を設ける（Phase 2〜6）
4. `conftest.py` に触らない — `db` fixture は Repository 設計と無関係に動く
5. API テスト内の生 Session 操作は移行対象にしない（テストが実装に結合するのを避ける）
6. Phase / ファイル単位でコミット — green な状態だけをコミット

## 6. 見落としやすいリスク・注意点

1. **`delete_user` のトランザクション原子性（最重要）** — 「Item の bulk delete → user の delete → commit 1 回」を Repository メソッドに機械的に分割すると、Item 削除後・User 削除前に失敗した場合に中途半端な状態が commit される
2. **`authenticate` の副作用（ハッシュアップグレード）** — bcrypt→argon2 への透過アップグレードで commit が走る。`get_by_email` だけ Repository 化して書き込み側を忘れると永続化されなくなる
3. **タイミング攻撃対策の `DUMMY_HASH`** — リファクタで「user が None なら即 return None」に簡略化するとセキュリティ対策が消える
4. **`read_user_by_id` の `user == current_user` 比較** — SQLAlchemy の identity map に依存。Repository が別 Session を使うと静かに壊れる
5. **`update_user_me` と `crud.update_user` は同じではない** — password→hashed_password 変換の有無が違う
6. **`private.py` の create は refresh していない** — `ENVIRONMENT == "local"` 限定ルートなので移行漏れと CI 未検出が起きやすい
7. **import 循環** — `repositories` から `api.deps` 方向への import は絶対に作らない
8. **count + list のペアの扱い** — Repository でのメソッド設計。`*sPublic` スキーマの組み立てを Repository に持ち込まない
9. **mypy / ruff の既存設定** — 型の改善は移行完了後に行う
10. **`initial_data.py` / 起動スクリプト経路** — pytest だけでなく Docker 起動経路も確認する

## 7. 完了の定義

- `app/crud.py` と `tests/crud/` が削除されている
- `app/` 配下で `session.add/commit/exec/get/delete` を直接呼ぶのは `app/repositories/` と `deps.get_db` のみ
- 全テスト・`mypy`・`ruff` が pass
- API の外形的挙動（レスポンス、ステータスコード、トランザクション境界）に変更がない
:::

### 計画書の品質評価

計画書をプロジェクトの実際のソースと照合して採点しました。

| 評価項目 | Fable 5 | Opus 4.8 | 確認方法 |
|---|---|---|---|
| 変更対象ファイルの網羅性 | ◯ | ◯ | `crud.py`・各ルート・`deps.py`・`core/db.py`・テストヘルパー全てを列挙しているか |
| 変更順序の妥当性 | ◯ | ◯ | Repository追加→委譲シム→ルート移行→テスト移行→削除という依存順か |
| テストへの言及 | ◯ | ◯ | `tests/crud/`・`tests/utils/`・`tests/api/routes/`それぞれへの対処を述べているか |
| リスク認識 | ◯ | ◯ | トランザクション境界・タイミング攻撃対策・Session同一性など各10項目 |
| 具体性 | ◯ | ◯ | ファイル名・関数名・フェーズ番号まで明示しているか |

両モデルとも全項目◯でした。見落としのあるファイルはなく、フェーズ設計の考え方も一致しています。

### 探索の進め方の違い

ツールコールのログとセッション内の発言ログを比べると、両モデルで明確なスタイルの差が出ました。

**Fable 5の動き方：**

まず `find` コマンドでバックエンド全体のファイル一覧を取得してから、`crud.py` → `deps.py` → `routes/users.py` → `routes/items.py` → `routes/login.py` → `routes/private.py` → `core/db.py` → `tests/conftest.py` → `tests/crud/test_user.py` → `tests/utils/user.py` → `tests/utils/item.py` と一気に11ファイルを読み込み、最後に `grep` でSession操作の所在を確認してから計画書を書きました。2回の試行で読んだファイルセットは全く同じで、**探索に迷いがありません**。完了後はシンプルに「計画書を作成しました」と要点サマリーを出力して終了です。

**Opus 4.8の動き方：**

同じコアファイルを読みつつ、Bash コマンドを4回使う点が異なります。run1では `backend_pre_start.py` という起動スクリプトまで確認し、`class` 定義の grep も追加しました。特徴的なのは、**計画書を書き終わった後に `ls` で書き込み確認 → 計画書を読み直して自己検証**するステップを踏んでいる点です。run2も `git log` でリポジトリの履歴を確認してから作業を開始し、同様に書いた計画書を最後に読み返しています。

さらに興味深い点がありました。今回のベンチマークでは4回のラン全てが同じリポジトリパスを使い、前のランが書いた計画書ファイルが残ったままになっていました。**Opus 4.8 は両ランとも「このパスに既に計画書が存在する」ことを認識し、自分のものではないと判断して内容を実コードと突き合わせて検証した上で報告しています。** Fable 5 はそのような言及を一切せず、ファイルを書き込んで終わりでした。

Opus 4.8 run1 の発言:
> "A plan file already exists (created 13:24, before this session). Per protocol I'll read it before overwriting."（意訳：13:24 作成の計画書が既にある。上書き前に読む）

Opus 4.8 run2 の発言:
> "バックエンドを独自に調査したところ、同じパスに既に充実した移行計画書が存在していました。私が書いたファイルではないので、勝手に上書きせず、まず実コードと突き合わせて内容が正しいか独自に検証しました。"

### 考察

**コストの差が想定以上に大きかった**

最も驚いた結果は品質でも速度でもなく、コストです。同じタスクで Opus 4.8 は Fable 5 の **52〜59%のコスト**で完了しました（run1: $1.09 vs $1.68、run2: $0.87 vs $1.48）。しかも品質は全評価項目で差がつかず、所要時間もほぼ互角です。

cache read トークンが Opus 4.8 で大幅に多い点（308k〜323k vs 214k〜227k）は、Fable 5 がより長いシステムプロンプトを内部に持ち、その分がキャッシュヒットに反映されている可能性を示唆しています。一方 output トークンは Opus 4.8 が多く（14k〜13k vs 12k〜11k）、より詳細な計画書を書いていました。それでも安い理由は、output トークン単価の違いと cache read の割引率が効いているためです。

**「読んで計画する」タスクはOpus 4.8の土俵**

今回のタスクは「コードを書かずに既存コードを読んで計画書を作る」という調査・整理型の作業です。コードを一行も変更しないため、Fable 5 が真価を発揮する「複雑な実装の意思決定」や「曖昧な要件の解釈」が求められる場面が少なく、結果として品質に差が出ませんでした。

探索スタイルでは Fable 5 が「迷いなく一直線」、Opus 4.8 が「確認しながら慎重に」という対比が見えましたが、どちらも同じファイルにたどり着いて同じ品質の計画書を出しています。

**Opus 4.8 の「既存成果物を確認してから動く」行動**

今回のベンチマーク設計上、Fable 5 が先に書いた計画書ファイルが Opus 4.8 の実行時も残っていました。Opus 4.8 は両ランともこれを発見し、上書き前に内容を実コードと突き合わせて検証する行動を取りました。Fable 5 はそのような確認をせず、ファイルの存在に言及することもありませんでした。

これは「コストが安い → 慎重で遅い」という単純な説明ではなく、**既存の文脈を重視するスタイルの違い**です。実際の開発現場でいえば、既存のドキュメントや設計書がある状態で作業させるとき、Opus 4.8 の方が「チームの既存成果物を尊重する」挙動を見せる可能性があります。

**使い分けの基準として**

このタスクを踏まえると、「調査・計画書作成・ドキュメント生成」のような読み取り重視のタスクでは Opus 4.8 の方がコスト効率が良い選択肢です。Fable 5 の出番は、実装する量が多い・要件が複雑・一発での完成度を求めるタスクに絞るのが合理的と感じました。

## 他の用途への応用アイデア

今回は改修計画タスクで比較しましたが、同じ計測アプローチは他のシーンにも使えます。

- **ドキュメント生成タスク**: 大規模なコードベースのREADMEやAPIドキュメントを生成させてトークン効率を比べる
- **バグ調査タスク**: エラーログと数十ファイルを渡して原因特定させる——探索の無駄が特に出やすいシーン
- **コードレビュータスク**: PRの差分を渡してレビューコメントを生成させて、見落としの少なさと冗長さを比べる

今回使ったベンチマークスクリプト（`claude --print --output-format stream-json --verbose`）はそのまま使い回せます。自分の普段使いタスクで同様の検証を試してみてください。

## まとめ

「既存のFastAPIプロジェクトをRepository パターンへ移行する計画書を作成する」というタスクで、Fable 5 と Opus 4.8 を各2回ずつ実測しました。

| | Fable 5 | Opus 4.8 |
|---|---|---|
| 所要時間 | 3分37秒〜3分54秒 | 3分28秒〜3分57秒 |
| コスト | $1.48〜$1.68 | $0.87〜$1.09 |
| 計画書の品質 | ◯ | ◯ |

結果として**品質・速度は同等で、コストはOpus 4.8が4割安い**という結果でした。

Fable 5 は「迷いのない直線的な探索」、Opus 4.8 は「書いた後に読み直す自己検証スタイル」という行動の違いはありましたが、最終成果物の差には結びつきませんでした。

「読んで計画する」タスクなら Opus 4.8 で十分です。Fable 5 の価値は、実装の複雑さや曖昧な要件への対処が要求されるタスクで発揮されます。Claude Code 上でのモデル選択は `/model` コマンドで即座に切り替えられるので、タスクの性質に応じて使い分けていくのが現時点でのベストプラクティスといえます。

## 参考リンク

- [Anthropic公式: Claude Fable 5 / Mythos 5発表](https://www.anthropic.com/news/claude-fable-5-mythos-5)
- [SWE-Bench Pro リーダーボード](https://labs.scale.com/leaderboard/swe_bench_pro_public)
- [FrontierCode: Introducing FrontierCode | Cognition](https://cognition.ai/blog/frontier-code)
- [FastAPI(GitHub)](https://github.com/fastapi/fastapi)
- [fastapi/full-stack-fastapi-template(GitHub)](https://github.com/fastapi/full-stack-fastapi-template)
