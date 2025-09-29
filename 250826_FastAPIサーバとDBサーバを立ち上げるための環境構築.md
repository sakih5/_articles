# FastAPI サーバと DB サーバを立ち上げるための環境構築

## 前提

- AWS 上での開発に移行することに備え、アプリ用コンテナとデータベース用コンテナを分けて作成する
  - compose.yml もアプリ用コンテナとデータベース用コンテナでそれぞれ作成する
- app は Dockerfile を作成してイメージを生成して、コンテナを作成する
- db は**既存のイメージをそのまま使用**して、コンテナを作成する

## フォルダ構成

```txt
/home/ユーザー名/projects/
└── backend-study_postgresql-test/
    ├── backend
    │   │   └── app # APIサーバのソースコード
    │   │       └── main.py
    │   ├── .dockerignore
    │   ├── Dockerfile
    │   └── requirements.txt
    ├── frontend # UIのソースコード
    │   ├── index.html
    │   └── main.js
    ├── .env
    ├── .env.example
    ├── .gitignore
    ├── compose.app.yml
    └── compose.db.yml
```

## 各ファイルの詳細

### `app`コンテナの`Dockerfile`

```dockerfile
# ベースイメージの設定
FROM python:3.12-slim

# OSパッケージの追加
RUN apt-get update && apt-get install -y --no-install-recommends build-essential && \
    rm -rf /var/lib/apt/lists/*

# 作業ディレクトリの指定
WORKDIR /app

# 環境変数の設定
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

# ライブラリのインストール
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# スクリプトフォルダ全体をコンテナ上にコピー
COPY app /app/app

# Uvicorn で起動（本番はホットリロードしないため --reload オプションはつけない。開発時はこのコマンドはcompose.ymlで上書きされる）
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]

# コンテナのポート番号の設定
EXPOSE 8000
```

#### 1. ベースイメージの設定

```dockerfile
FROM python:3.12-slim
```

- `python:3.12-slim`は、Docker Hub で公開されている Python 公式イメージの 1 つ

  - 軽量な Debian linux 上に既に Python がインストールされたもの
  - `pip`も使えるようになっている
  - セキュリティ対策も万全

- `FROM (イメージ名)`と書くことで、この"言語ランタイム"Dockerfile を直接いじるのではなく、**自分のアプリ用 Dockerfile で継承**することができる（それが定石）

  - `FROM`で指定したイメージは**ベースイメージ**と呼び、Docker イメージの**一番下のレイヤー**のことを指す

#### 2. OS パッケージの追加

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends build-essential && \
    rm -rf /var/lib/apt/lists/*
```

- このコマンドの概略: `python:3.12-slim`には`build-essential`はインストールされていないので、追加でインストールする の意
- 追加インストールするパッケージ`build-essential`の説明:
  - Debian/Ubuntu 系で「C/C++ の開発環境一式」をまとめて入れるためのメタパッケージ
  - `build-essential`の中に、`gcc`（GNU C コンパイラ）・`g++`（GNU C++コンパイラ）・`make`（ビルド自動化ツール）・`libc6-dev`（標準 C ライブラリ（glibc）のヘッダや開発用ファイル）・`dpkg-dev`（Debian パッケージ開発ツール群）が入っている
  - `python:3.12-slim`には`build-essential`は入っていないが、その中身の`gcc`や`make`、`libc6-dev`、`dpkg-dev`は一度入っている。でも`g++`は入っていないし、**ほかのパッケージもビルド後削除されてしまっている等する**ので、`numpy`や`pandas`を使うのであれば入れなおしておいた方がいい
- インストール時のオプション:

  - `--no-install-recommends`: 依存関係にない推奨パッケージは入れないという意（パッケージを軽くする）
  - `rm -rf /var/lib/apt/lists/*`: apt のキャッシュを削除してさらに軽量化という意

- `(コマンド1) && (コマンド2)`は、コマンド 1 を実行して成功したらコマンド 2 を実行するの意

#### 3. 作業ディレクトリの指定

```dockerfile
WORKDIR /app
```

- コンテナ内の作業ディレクトリを/app に設定
- 以降の命令（`COPY`、`RUN`、`CMD`）は、**カレントディレクトリを`/app`にして実行される**

#### 4. 環境変数の設定

```dockerfile
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1
```

- `PYTHONDONTWRITEBYTECODE=1`: Python が`.pyc`ファイル（バイトコードキャッシュ）を生成しないようにする

  - `.pyc`は通常、実行速度を少し上げるために **pycache** フォルダに作られる
  - Docker コンテナの中では

    - 永続化しない（コンテナ再作成で消える）
    - 層が汚れてイメージが大きくなる

    → 無駄が多いので生成禁止にするのがベストプラクティス

- `PYTHONUNBUFFERED=1`: Python の出力を バッファリングせずに即時フラッシュする

  - デフォルトだと stdout/stderr はバッファに貯めてまとめて出力 → Docker のログにすぐ反映されないことがある
  - これを有効にすると print() の内容が即座にログに出るので、デバッグ・運用でログが遅延しない

- `PIP_NO_CACHE_DIR=1`: `pip`がパッケージをインストールした後のキャッシュを残さない

  - 通常`pip`は`~/.cache/pip`にダウンロードした`wheel`や`tarball`を保存する
  - Docker ビルドではキャッシュは再利用されないし、ただイメージが膨らむ原因になる

  → そのためキャッシュを無効化してイメージサイズを節約する

#### 5. ライブラリのインストール

```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

- `COPY requirements.txt .`: `requirements.txt`をコンテナのカレントディレクトリ（`/app`）直下にコピー
- `COPY <ホスト側のパス> <コンテナ側のパス>`の意
  - `<ホスト側のパス>` = `docker build`を実行したときの「ビルドコンテキスト」（通常は Dockerfile があるディレクトリ）を基準にしたパス
  - `<コンテナ側のパス>` = イメージの中でのコピー先パス
- `RUN pip install --no-cache-dir -r requirements.txt`: `requirements.txt`に書かれたライブラリをインストール

  - `-r <ファイルパス>`: ファイルパスのファイルに列挙されたライブラリをインストールする`-r`は**requirement file**の意
  - `--no-cache-dir`: pip が使うキャッシュ（`~/.cache/pip`）を作らない・使わない

    - 通常は、`pip`はダウンロードしたパッケージをキャッシュに保存する
    - Docker ビルドではこのキャッシュは 次のビルドで再利用されない ＋ イメージに残ってサイズを膨らませる
    - そこで`--no-cache-dir`を付けて「キャッシュ残すな」と指示

    → 結果、軽量なイメージが作れる

#### 6. スクリプトフォルダ全体をコンテナ上にコピー

```dockerfile
COPY app /app/app
```

- ローカルにあるフォルダを中身のファイル丸ごと、ビルド中にイメージへ取り込む

  - Dockerfile でできる設定だと、イメージを生成するタイミングのコードをそのままコンテナに積む感じになってしまう

    → 開発中コードのリアルタイムでの反映・更新は、Dockerfile ではなく、`compose.yml`の**バインドマウント**で設定する

  - 一方で、本番時は`compose.yml`でのバインドマウントは行わず、`COPY`で確定コードをイメージに焼き込んでコンテナにして使う

#### 7. Uvicorn で起動（本番はホットリロードしないため --reload オプションはつけない。開発時はこのコマンドは compose.yml で上書きされる）

```dockerfile
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

- コンテナ起動時に実行されるコマンド
- `app`フォルダにある`main.py`ファイルの中にある`app`インスタンス（FastAPI クラスをインスタンス化したオブジェクトの変数名）を ASGI アプリケーションとして Uvicorn（ASGI サーバ）上で起動する。以降、リクエストが来たら ASGI プロトコルを通して`app`が処理する
- コンテナの待ち受けアドレスを`0.0.0.0:8000`とする
  - `0.0.0.0`（ネットワークアドレス）にしておくことで、コンテナの外（ホスト PC や他のコンテナ）からアクセスできる
    - 「コンテナ上の全ての NIC」で待ち受ける、の意
    - `127.0.0.1`（ループバック・localhost）を設定してしまうと、コンテナの中でしかアクセスできなくなる
  - サーバプログラム（Uvicorn + FastAPI）はネットワーク上の特定のアドレスとポートに「耳をすませている」状態となる。このことを**ソケットを listen する（待ち受ける）**という。誰か（ブラウザや curl）がリクエストを送ってきたら、それを受け取って処理する準備ができている状態

#### 8. このコンテナのポート番号を指定

```dockerfile
EXPOSE 8000
```

- 「コンテナが 8000 番ポートを開く」という宣言（ドキュメント的な意味合い）
- 実際の公開は Compose 側`ports: "8000:8000"`で行われる

### `compose.app.yml`

- FastAPI アプリケーションを Docker Compose で動かす設定

```yml
services:
  app: # appというサービスを定義
    build: # イメージを./backend/Dockerfileからビルド
      context: ./backend
      dockerfile: Dockerfile
    env_file: .env # 環境変数は.envファイルから読み込む
    ports: # 左がホストのポート、右がコンテナのポート
      - '8000:8000'
    volumes: # ソースコードをホストとコンテナで同期（ボリュームマウント）
      - ./backend/app:/app/app:rw
    command: >
      sh -c "uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload"
```

#### フォルダマウント

```yml
volumes: # ソースコードをホストとコンテナで同期（ボリュームマウント）
  - ./backend/app:/app/app:rw
```

- ホスト側`./backend/app`をコンテナ内`/app/app`に`rw`（読み書き可能な形で）マウント（同期）する
- これがないと、コンテナは「ビルド時点のコードのコピー」しか持たないので、ホストでコードを直してもコンテナには伝わらない
- ファイルの変更がコンテナに届く仕組みを作っているのがこの部分

#### Uvicorn アプリの立ち上げ

```yml
command: >
  sh -c "uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload"
```

- `>`は YAML の書き方（ブロックカラースタイル）。`>`をつけると「改行をスペースに置き換えて、1 行の文字列として扱う」というルールになる
- `>`を使う理由

  - コマンドが長くなると、YAML ファイル内で横に伸びて読みにくい
  - `>`を使うと 見やすく改行して書ける
  - YAML 的には 1 行にまとめたのと同じ扱いになる

    ```yml
    command: sh -c "uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload"
    ```

- `--reload`オプションをつけることで、コードの中身の変更を自動で検知してサーバを再起動（リロード）する
- 同じコマンドが Dockerfile にも記載あるが、`compose.yml`にある内容で上書きされる

### `.dockerignore`

```txt
.git
.venv
__pycache__
*.pyc
node_modules
.DS_Store
```

- このファイルに記載した内容は、イメージを生成する時のみ使われる
- `.dockerignore`に書かれたファイルやフォルダは、ビルドコンテキスト（ホストから Docker デーモンに送られる一式）に含まれなくなり、ビルド速度が速くなる
- 「不要なファイルをイメージに含めないためのフィルタ」

### `requirements.txt`

```txt
fastapi==0.111.0
uvicorn[standard]==0.30.1
pydantic==2.6.4
psycopg[binary]==3.2.1
python-dotenv==1.0.1
```

- `fastapi`: フレームワーク本体（ルーティング、バリデーション、API 仕様自動化）
- `uvicorn`: 実際にアプリを走らせるサーバー＋便利な高速化・開発補助ライブラリ一式
- `pydantic`: データモデル定義と検証を行うライブラリ。定義しておくと、FastAPI が自動で入力チェック（バリデーション）やレスポンス整形をしてくれる
- `psycopg[binary]`: Python から PostgreSQL に接続するための公式ドライバ。`binary`を付けると、C 拡張を含んだバイナリ版がインストールされるため、ビルド環境がなくてもそのまま使える（Docker 上でも楽に動作する）
- `python-dotenv`: `.env`ファイルに書いた環境変数をアプリ実行時に読み込むライブラリ。例えば DATABASE_URL や POSTGRES_USER をコードから簡単に利用できるようになる

### `compose.db.yml`

```yml
services:
  db: # dbというサービスを定義
    image: postgres:16 # ビルドするイメージの指定
    env_file: .env # 環境変数の読み込み
    healthcheck: # コンテナが健康に起動しているかチェックする設定
      test: ['CMD-SHELL', 'pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB']
      interval: 5s
      timeout: 5s
      retries: 10
    ports: # ホストの 5432 をコンテナの 5432 にフォワード
      - '5432:5432'
    volumes: # データの永続化
      - pg_data:/var/lib/postgresql/data

volumes: # ボリュームの定義
  pg_data:
```

#### ビルドするイメージの指定

```yml
image: postgres:16
```

- Docker Hub の公式 PostgreSQL イメージ（バージョン 16）を使う
- 自分でイメージをビルド（作成）しないため、**db 用の Dockerfile は無い**

#### 環境変数の読み込み

```yml
env_file: .env
```

- 以下の環境変数の情報を読み込んで、コンテナが作成される
  - `POSTGRES_USER`を渡すと、その名前でスーパーユーザーを作成（デフォルトは “postgres”）
  - `POSTGRES_PASSWORD`はスーパーユーザーのパスワード（ここは省略してはいけない）
  - `POSTGRES_DB`を渡すと、その名前のデータベースを作成（省略時は POSTGRES_USER 名と同じになる）
- 上記環境変数名は、Docker Hub の公式 PostgreSQL イメージ（バージョン 16）で説明されている

#### コンテナが健康に起動しているかチェックする設定

```yml
healthcheck:
  test: ['CMD-SHELL', 'pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB']
  interval: 5s
  timeout: 5s
  retries: 10
```

- `pg_isready`は PostgreSQL が接続可能か調べるコマンド
- `-U`でユーザー指定、`-d`でデータベース指定
- `$$POSTGRES_USER`は`.env`の変数を参照する書き方（`$`を YAML 上でエスケープするため`$$` と書いている）
- 結果が OK なら「healthy」と判定
- `interval: 5s` → 5 秒ごとにチェック
- `timeout: 5s` → 応答が 5 秒以内になければ失敗
- `retries: 10` → 10 回連続失敗したら「unhealthy」とみなす

#### データの永続化

```yml
volumes:
  - pg_data:/var/lib/postgresql/data
```

- コンテナ内の`/var/lib/postgresql/data`（DB のデータ保存場所）を、ホストのボリューム `pg_data`に永続化
- コンテナを削除しても、データは`pg_data`に残る

#### ボリュームの定義

```yml
volumes:
  pg_data:
```

- 名前付きボリューム`pg_data`を定義
- ここで名前だけ宣言すれば、Docker が自動で保存場所を管理してくれる

### `.env`

```env
# for postgres
POSTGRES_USER=appuser
POSTGRES_PASSWORD=secret
POSTGRES_DB=appdb

# for app
DATABASE_URL=postgresql+psycopg[binary]://appuser:secret@db:5432/appdb
```

- 各自動作確認を行う際に、`env.example`をコピーして、`.env`を作成する

## 動作確認

1. 複数ファイル指定でまとめて起動する

```bash
docker compose -f compose.db.yml -f compose.app.yml up -d --build
```

- 起動時に複数ファイル指定でまとめて起動することにより、同一プロジェクトとして扱われ、同じデフォルトネットワークにのる

2. DB のヘルス確認

- db コンテナが (healthy) になっていることを確認
- もし`starting`が続く場合はログ確認

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

## Docker の基本知識

- Docker Engine をインストールすることで、コンテナを作成したり使ったりすることができる
- Docker Engine の上に、コンテナを載せる。コンテナは容量が許す限りいくつでも載せられる
- Docker Engine はソフトウェアである。起動や停止ができる。起動させておくことでコンテナを作成したり使ったりすることができる
- コンテナはイメージから作られる
- イメージは金型のようなもので、イメージが 1 つあればコンテナは何個でも作ることができる
- 逆にコンテナからイメージを作ることもできる
- Docker Engine は Linux OS 上にしか載せられない。Windows PC の場合は WSL をインストールして、その上に載せることが必要
- コンテナの中には、「Linux OS の周辺機能（要は Linux カーネル以外の部分）」が入っている
  - なぜなら、コンテナの中に入っているプログラムが指示したことを「Linux OS の周辺機能」がキャッチするには、コンテナの中に「Linux OS の周辺機能」を入れておく必要があるため
- Docker では、コンテナの外にある Linux カーネルを使う

## Dockerfile の基本知識

- Dockerfile は、Docker イメージを作るための設計図（テキストファイル）

  1. `docker build`コマンドで Dockerfile → Docker イメージを生成
  2. `docker run`コマンドで Docker イメージ → コンテナを生成
  3. `docker start`コマンドでコンテナを立ち上げる

  ```txt
  Dockerfile
  ↓ docker build
  Docker イメージ
  ↓ docker run
  コンテナ
  ```

- Docker イメージは階層構造になっている。Dockerfile はベースとなる層の設定から順番に書く

  1. どの OS を土台にするか？
  2. どんなパッケージを入れるか？
  3. アプリをどう配置するか？
  4. アプリをどう起動するか？

### 補足 コンテナの再利用の有無について

- アプリ用コンテナは再利用しない。毎回 Docker イメージからコンテナを生成する
- データベース用コンテナは開発中はコンテナを再利用することもある
