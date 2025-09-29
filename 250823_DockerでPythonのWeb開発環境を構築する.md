## 前提

- Windows PC を使用
- WSL2 をインストールしており、そこに Docker Engine もインストール済
- Docker Desktop はインストールしていない
- FastAPI で API サーバを構築するための環境構築をしたい

## 大まかな流れ

1. 以下のようなディレクトリ構成とする

   ```txt
   /home/ユーザー名/projects/
   └──  backend-study_fastapi-test/
       ├── app
       │   └── main.py    # ソースコード
       ├── Dockerfile
       ├── requirements.txt
       └── compose.yml
   ```

2. Dockerfile、requirements.txt、compose.yml を書く
3. app/main.py を書く
4. Dockerfile から Docker イメージを作成する
5. Docker イメージからコンテナを作成する
6. コンテナ上に立ち上がった FastAPI サーバにアクセスする

## 実際の手順

### 1. Dockerfile を書く

- Dockerfile とは？

  - Docker イメージを作るための設計図（テキストファイル）
  - 「どの OS を土台にする？」「どんなライブラリを入れる？」「アプリをどう配置して、どう起動する？」を順番に書いていく
  - Dockerfile を`docker build`すると、指示どおりにレイヤーを積み上げた**Docker イメージ**が出来上がる
  - そして`docker run`すると、**Docker イメージ**からコンテナを起動することができる

  - アプリ用コンテナは毎回 Docker イメージからコンテナを生成するのが普通。一方で、DB 用コンテナは開発中はコンテナを再利用する(`docker stop`、`docker start`)こともある
  - ケーキで例えると…
    - `run` = ホールケーキから新しい一切れを切る（毎回新しいコンテナをイメージから作る）
    - `stop` = 一切れを冷蔵庫にしまう（消えない）（コンテナは削除されず、停止状態で残る）
    - `start` = 冷蔵庫から出してまた食べ始める（停止したコンテナを再度動かす）
    - `rm` = 一切れを捨てる（コンテナを削除する）

- 今回の記載例：ひと言でいうと、「Python 入りの Linux に FastAPI をインストールして、uvicorn で起動するイメージ」

  ```dockerfile
  # ベースイメージの設定
  FROM python:3.12-slim

  # OSパッケージの追加
  RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential && rm -rf /var/lib/apt/lists/*

  # 作業ディレクトリの指定
  WORKDIR /app

  # 環境変数の設定
  ENV PYTHONDONTWRITEBYTECODE=1 \
  PYTHONUNBUFFERED=1 \
  PIP_NO_CACHE_DIR=1

  # 依存関係
  COPY requirements.txt .
  RUN pip install --no-cache-dir -r requirements.txt

  # アプリ本体
  COPY app ./app

  # Uvicorn で起動（本番は --reload しない）
  CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
  EXPOSE 8000
  ```

  - `FROM python:3.12-slim`: このコンテナは「何を土台にするか」の命令。Docker イメージはレイヤー構造になっていて、一番下のレイヤーを決めるのが`FROM`。これをベースイメージという。以降の`RUN`や`COPY`はこのベースイメージの上に積み上げていくイメージ。また、`python:3.12-slim`は、Docker Hub で公開されている公式 Python イメージの 1 つであり、この開発チームがメンテをしているためセキュリティアップデートなどの対応をしてくれるため安心。実体は、軽量な Debian Linux 上に既に Python がインストールされたもの
  - `RUN apt-get update && apt-get install -y --no-install-recommends build-essential && rm -rf /var/lib/apt/lists/*`: パッケージリストを更新した上で、ビルド環境に必要なツールをインストールする

    - ここでいうビルド環境とは？
      → **Python ライブラリなどをインストールするときに裏で必要になる C 言語コンパイラや関連ツールが揃った環境のこと**）
    - `--no-install-recommends`は依存関係にない「推奨パッケージ」を入れないという意（パッケージを軽くする）。
    - `rm -rf /var/lib/apt/lists/*` → apt のキャッシュ削除してさらに軽量化という意

  - `WORKDIR /app`: コンテナ内での作業ディレクトリを`/app`に設定。以降の命令（`COPY`、`RUN`、`CMD`）は、カレントディレクトリを`/app`にして実行される
  - `COPY requirements.txt .    RUN pip install --no-cache-dir -r requirements.txt`: 依存ファイルをコピー・ライブラリをインストール

    - `requirements.txt`をコンテナのカレントディレクトリ（`/app`）直下にコピー

      ```dockerfile
      COPY <ホスト側のパス> <コンテナ側のパス>
      ```

      - ホスト側のパス = docker build を実行したときの「ビルドコンテキスト」（通常は Dockerfile があるディレクトリ）を基準にしたパス
      - コンテナ側のパス = イメージの中でのコピー先パス

    - `pip install`で必要なライブラリをインストール
    - `--no-cache-dir` → pip のキャッシュを残さずに消してサイズ削減
    - **依存関係のコピー＆インストールを「アプリ本体より前」にするのがポイント**。コードを少し直すたびにライブラリを全部再インストールしないで済む（Docker のビルドキャッシュを効かせるため）

  - `COPY app ./app`: 自分が書いた FastAPI アプリ (app/ フォルダ) をコンテナにコピー
  - `CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]`: コンテナ起動時に実行されるコマンド

    - コンテナの待ち受けアドレスを`0.0.0.0:8000`とする
    - サーバプログラム（Uvicorn + FastAPI）はネットワーク上の特定のアドレスとポートに「耳をすませている」状態となる。このことを**ソケットを listen する（待ち受ける）**という。誰か（ブラウザや curl）がリクエストを送ってきたら、それを受け取って処理する準備ができている状態

  - `EXPOSE 8000`: 「コンテナが 8000 番ポートを開く」という宣言

- インストールする Python のライブラリを requirements.txt に記載する

  ```txt
  fastapi
  uvicorn[standard]
  ```

  - それぞれの意味:

    - fastapi: フレームワーク本体（ルーティング、バリデーション、API 仕様自動化）
    - uvicorn: 実際にアプリを走らせるサーバー＋便利な高速化・開発補助ライブラリ一式

- 今回は Dockerfile に加えて compose.yml ファイルを作成する

  - compose.yaml ファイルでは、Dockerfile で定義されたイメージを含むサービス全体の起動方法が定義される

  ```yaml
  services:
  api:
    build: .
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    volumes:
      - ./app:/app/app:rw
    ports:
      - '8000:8000'
    environment:
      - PYTHONUNBUFFERED=1
    # Linux/WSL なら不要だが、権限が気になるときは下記を使う手も
    # user: "${UID:-1000}:${GID:-1000}"
  ```

  - それぞれの意味:

    - `build: .`: このディレクトリにある Dockerfile を使ってイメージを作る
    - `command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload`: 「Uvicorn という ASGI サーバーを使って、FastAPI アプリを指定して起動する」命令

      - `uvicorn app.main:app`: `app/main.py`の中の`app = FastAPI()`（FastAPI アプリ）を起動する、の意
      - `--reload`: ソースコードを変更すると自動で再起動（開発用）← **ホットリロード**という
      - `--host 0.0.0.0`: コンテナ内で、どの IP アドレスで待ち受けるか。`0.0.0.0`にすることで、すべてのネットワークインターフェース（同じ LAN 内の別 PC やスマホなど）からアクセスできるようになる。※Docker や WSL 上で FastAPI を動かすときは、`0.0.0.0`にしないとホスト PC からアクセスできないことが多い

        → 要は、**ホスト PC から FastAPI アプリにアクセスできる**ようにするために、このオプションは必要

      - `--port 8000`: `http://localhost:8000` でアクセス

    - `volumes: - ./app:/app/app:rw`: ホスト側の ./app ディレクトリを、コンテナ内の /app/app にマウントすることで、**ローカルでのコード変更をリアルタイムでコンテナに反映する**

      - 左側の`./app`はホスト PC（WSL）で Dockerfile と同じ階層（カレントディレクトリ）にある app フォルダのことを指す
      - 右側の`/app/app`は、コンテナの中のディレクトリ
      - `:rw`は read/write（読み書き可）

    - `ports: - "8000:8000"`: 左側の`8000`はホスト側（WSL）、右側の`8000`はコンテナ側のポート

      → 　つまり、コンテナの中で FastAPI が`0.0.0.0:8000`で待ち受けているものを、ホスト PC の`localhost:8000`に公開している。そのため、ブラウザから`http://localhost:8000`で FastAPI サーバにアクセスできる

    - `environment: - PYTHONUNBUFFERED=1`: Python の出力を即時出力にする（バッファリングしない）

  - 補足: Dockerfile と compose.yml のすみ分け

    - Dockerfile の役割: イメージ（完成品テンプレート）を作るためのレシピ

      - どの OS／ベースイメージを使うか (FROM python:3.12-slim)
      - どんなパッケージをインストールするか (pip install …)
      - どのファイルをコンテナにコピーするか (COPY …)
      - コンテナが起動したときに実行するコマンド (CMD …)

      → 「アプリ入りの箱をどう作るか」 を記述する。結果として fastapi-app:latest のような イメージが出来上がる。

    - compose.yaml の役割: そのイメージをどう使ってサービスを動かすか を定義するファイル

      - どのイメージを使うか／どこからビルドするか (build: .)
      - 起動コマンドを開発用に差し替える (command: uvicorn … --reload)
      - ホスト PC とコンテナをどうつなぐか (ports: 8000:8000)
      - コードをどう共有するか (volumes: ./app:/app/app)
      - 環境変数をどう渡すか (environment: …)
      - さらに、DB やキャッシュサーバーなど複数サービスを一括で管理できる

      → 「イメージをどう配置し、どう繋いで、どう起動するか」 を記述する。

### 2. Dockerfile から Docker イメージを作成する

- 以下コマンドで、

  1. compose.yml に記載された、「どの Dockerfile を、どんな条件でビルドするか」の指示をもとに
  2. イメージのビルド（作成）を Dockerfile を使って行う

  ```bash
  docker compose build
  ```

- 出来上がったイメージ名は、自動で`ディレクトリ名_サービス名:latest`となる。今回の場合、ディレクトリ名は`backend-study_fastapi-test`、サービス名は`api`（Dockerfile に記載されている）なので、`backend-study_fastapi-test_api`となる

- イメージ名は`docker image ls`で確認できる（上記確認してみたら`backend-study_fastapi-test-api`と出た。区切り文字が「\_」ではなく「-」となっている。なぜかは分からない）

### 3. Docker イメージからコンテナを作成し、起動する

- 以下コマンドを実行する

  ```bash
  docker compose up
  ```

- `docker compose up`でビルドからコンテナ起動までやれてはしまうが、ビルドエラーにならないかチェックするためにも事前に`docker compose build`でビルドできるかチェックだけしておいている
- `docker compose up`で、Docker イメージが無い場合はイメージの作成からしてくれるが、Docker イメージがある場合はそのままそれを使ってコンテナを起動してくれる

### 4. FastAPI サーバにアクセスする

- その 1: ブラウザで`http://localhost:8000/`にアクセス

  - ここで見れるのは JSON そのもの
  - FastAPI には自動ドキュメント生成機能がある。ドキュメントの形式は Swagger UI か ReDoc か選べる

- その 2: ターミナル上で`curl http://localhost:8000/`を実行（`curl`はリクエストを送るコマンド、の意）

### 5. コンテナを停止する

`docker compose up`をフォアグラウンドで動かしているので、`Ctrl + C`を押すことで、SIGINT を送りコンテナを停止させることができる。

### 6. 停止したコンテナを再度動かす

- Dockerfile があるディレクトリに移動してから、`docker compose up`を実行する
- コンテナは再利用しないの？ → 原則しない

```bash
cd /home/sakih/projects/backend-study_fastapi-test
docker compose up
```
