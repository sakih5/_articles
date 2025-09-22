# FastAPI サーバと DB サーバを立ち上げるための環境構築

## 前提

- AWS 上での開発に移行することに備え、アプリ用コンテナとデータベース用コンテナを分けて作成する
  - compose.yml もアプリ用コンテナとデータベース用コンテナでそれぞれ作成する
- app は Dockerfile を作成してイメージを生成して、コンテナを作成する
- db は**既存のイメージをそのまま使用**して、コンテナを作成する

## フォルダ構成

## 各ファイルの詳細

### `app`コンテナの`Dockerfile`

```dockerfile
# ベースイメージの設定
FROM python:3.12-slim

# OSパッケージの追加
RUN apt-get update && apt-get install -y --no-install-recommends build-essential && \
    rm -rf /var/lib/apt/lists/*

# 作業ディレクトリ
WORKDIR /app

# 環境変数の設定
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

# ライブラリのインストール
COPY requirements.txt .
RUN pip install -r requirements.txt

# アプリ本体
COPY app /app/app

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
- インストール時のオプション:

  - `--no-install-recommends`:

- `build-essential`は
- `(コマンド1) && (コマンド2)`は、コマンド 1 を実行して成功したらコマンド 2 を実行するの意

### `compose.app.yml`

## Docker の基本知識

- Docker Engine をインストールすることで、コンテナを作成したり使ったりすることができる
- Docker Engine の上に、コンテナを載せる。コンテナは容量が許す限りいくつでも載せられる
- コンテナはイメージから作られる
- Docker Engine は Linux OS 上にしか載せられない。Windows PC の場合は WSL をインストールして、その上に載せることが必要
- コンテナの中には、「Linux OS の周辺機能（要は Linux カーネル以外の部分）」が入っている。コンテナの中に入っているプログラムが指示したことを「Linux OS の周辺機能」がキャッチするには、コンテナの中に「Linux OS の周辺機能」を入れておく必要がある

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
