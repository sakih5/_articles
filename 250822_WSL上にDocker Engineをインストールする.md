# WSL上にDocker Engineをインストールする

## 前提

Dockerのインストール方法は2つある。

- Docker Desktop（DockerのGUIアプリ）をインストールする
- Docker Desktopをインストールしない（代わりにCUIであるDocker Engineのみインストールする）

Docker Desktopは商用利用かつ、ある程度の規模の会社で行う場合は**有料**となる。そのため今回はDocker Desktopを使わずにDockerを利用するやり方を試してみる。

## 手順

### 1. 前提

- Windows PCにWSLをインストールした後、Ubuntu24.04をインストール済み
- Docker Desktopは入れていない（アンインストールした）

### 2. WSLのsystemdを有効化

- 目的：WSLでは、デフォルトでsystemdが無効になっており、そのままだとdockerが使いづらいため、systemdを有効化する

- 以下、手順。まず、WindowsのPowerShellではなく、Ubuntu側で以下コマンドを実行（WSLの設定ファイルをnanoエディタで編集する、の意）

```bash
sudo nano /etc/wsl.conf
```

- 内容をこうする（何も書いていなければ新規でベタ貼りする）

```ini
[boot]
systemd=true
```

- 保存後、WindowsのPowerShellでWSLを再起動して、設定を反映させる

```PowerShell
wsl --shutdown
```
### 3. 古いDockerが残っていれば削除

- Docker関連の以下のパッケージがあれば削除

  - docker
  - docker-engine
  - docker.io
  - containerd
  - runc

- コマンドは以下

  ```bash
  sudo apt-get remove -y docker docker-engine docker.io containerd runc || true
  ```

  - `|| true`は、もし左の処理でエラーになっても、成功終了するの意（`true`は必ず成功終了するという意のコマンド）

- 実行結果の例は以下。

  ```bash
  Reading package lists... Done
  Building dependency tree... Done
  Reading state information... Done
  Package 'docker.io' is not installed, so not removed
  E: Unable to locate package docker
  E: Unable to locate package docker-engine
  ```

  - 上3行は、`apt`がパッケージ情報を読み込み終わったという意
  - docker.io パッケージはシステムに入っていなかったので、削除対象がなくスキップされた
  - docker と docker-engine という名前のパッケージは、そもそもパッケージリストに存在しないので削除できなかった（古い Ubuntu では存在していたが、現在の Ubuntu 24.04 には最初から無い）

- 補足：パッケージリストに存在しないってどういう意味？

  → apt が知っているソフトウェアの一覧にその名前が載っていない。よって、インストールも削除もできないということ。
  
  Ubuntu 24.04には`docker`や`docker-engine`というパッケージがそもそも存在しない。昔の Ubuntu にあったパッケージ名が、廃止されているのだ。代わりに`docker.io`がUbuntu標準リポジトリにある

### 4. セキュリティ関連パッケージをインストールする

- 以下コマンドを実行

  ```bash
  sudo apt-get update
  sudo apt-get install -y ca-certificates curl gnupg
  ```

- 各パッケージの役割は以下

  1. ca-certificates
  - HTTPS 通信でサーバー証明書を検証するために必要な「認証局(CA)証明書」のセット
  - これがないと、https://download.docker.com/ のような HTTPS リポジトリに安全にアクセスできない

  2. curl
  - コマンドラインでデータを取得するツール
  - 後で Docker の GPG 鍵やリポジトリ設定ファイルをダウンロードするのに使う

  3. gnupg
  - ダウンロードした GPG 鍵を管理・検証するためのツール
  - APT がリポジトリから取得するパッケージが「Docker公式が署名したもの」だと確認するのに必須

- 補足：GPGキーとは？

  GNU Privacy Guard keyの略。ファイルや通信が「改ざんされていない」「正しい相手から届いた」ことを確認するための 暗号化の仕組みに使う鍵

  Docker公式は
  - パッケージにデジタル署名をつけていて
  - 検証できるように公開鍵（GPGキー）も配布している

  → `curl`コマンドでGPGキーをダウンロードし、`gnupg`コマンドでGPGキーを管理・デジタル署名との整合性チェックを行っている

- 実行結果は以下

  ```bash
  Reading package lists... Done
  Building dependency tree... Done
  Reading state information... Done
  ca-certificates is already the newest version (20240203).
  ca-certificates set to manually installed.
  curl is already the newest version (8.5.0-2ubuntu10.6).
  curl set to manually installed.
  gnupg is already the newest version (2.4.4-2ubuntu17.3).
  gnupg set to manually installed.
  0 upgraded, 0 newly installed, 0 to remove and 4 not upgraded.
  ```

  - ca-certification、curl、gnupgいずれも最新のものがインストール済み、かつ、手動インストールのフラグがついているの意
  - 何も新しくアップグレード、インストール、削除されたものはない、今回アップグレードできるものが4つ見つかったけど、指定はされていないのでしていないの意

- 補足：手動インストールのフラグとは？

  APT にはパッケージに対して次の2種類の「インストール状態のフラグ」がある

  1. 手動 (manual) インストール
    - ユーザーが直接 apt-get install パッケージ名 で入れたと扱われる
    - 「重要だから残しておこう」という扱いになる
    - 不要パッケージの自動削除 (apt autoremove) の対象にならない

  2. 自動 (auto) インストール
    - 依存関係として一緒に入っただけのもの
    - もし依存元のパッケージが削除されたら、autoremove で「もういらないね」と消される可能性がある

  ca-certificationやcurlは、Ubuntu24.04に予め入っていたものであり、手動で入れたわけではないが、重要なので**手動インストールのフラグ**がついている

### 5. DockerのGPGキーをダウンロードし、配置する

- 以下コマンドを実行して、GPGキーを置くディレクトリを作成

  ```bash
  sudo install -m 0755 -d /etc/apt/keyrings
  ```

  - `install`コマンドを使うことで、`mkdir`と`chmod`がいっぺんにできる

- GPG公開鍵（https://download.docker.com/linux/ubuntu/gpg）をダウンロード

  ```bash
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  ```

  - GPG公開鍵をダウンロード、その際のオプションは以下
    - `-f` → エラー時は何も出さず終了
    - `-s` → 進捗やメッセージを非表示（サイレント）
    - `-S` → エラーがあれば表示（-sと一緒に使う）
    - `-L` → リダイレクトを追跡
  - その鍵を`gpg --dearmor`で「APTが読める形式（バイナリ形式）」に変換して保存
  - 保存先は`/etc/apt/keyrings/docker.gpg`

- GPG公開鍵を誰でも（所有者、所有グループ、それ以外全て）読み取り可能にする

  ```bash
  sudo chmod a+r /etc/apt/keyrings/docker.gpg
  ```

- APTにDockerの公式リポジトリを登録する

  ```bash
  echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
    https://download.docker.com/linux/ubuntu \
    $(. /etc/os-release; echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  ```

  - 大まかな流れは以下。
    1. `echo`でリポジトリ設定の1行※を作る
    2. その結果を`sudo tee`でファイルに書き込む
    3. `> /dev/null`でコマンドラインへの出力は無しにする

  - ※1行の例：

    ```ini
    deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable
    ```

    - 意味：
      - deb → Debian形式のパッケージを使う
      - arch=amd64 → CPUアーキテクチャは amd64 用
      - signed-by=... → 署名の検証はこの鍵で
      - https://download.docker.com/linux/ubuntu → リポジトリのURL
      - noble → Ubuntuのコードネーム（24.04の場合）
      - stable → 安定版パッケージ群

  - APTのリポジトリ設定ファイルについて

    → 以下の2種類ある。いずれも「どこからパッケージを取ってくるか」が書かれている

    - メインの設定 → /etc/apt/sources.list
    - 個別の追加リポジトリ → /etc/apt/sources.list.d/*.list

### 6. 新しいDockerをインストールする

- 以下コマンドを実行

  ```bash
  sudo apt-get update
  sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
  ```

  - 意味：`sudo apt-get update`で、APTのリポジトリ設定ファイルをもとに「いま入手可能なパッケージ一覧（パッケージリストともいう）」を更新する
  - ※ APTのリポジトリ設定ファイルを編集したので、その反映のためにこのコマンドは飛ばしてはいけない
  - 各パッケージの意味:
    - docker-ce: CEは Community Edition の略。Docker本体のエンジン（dockerdデーモン）。コンテナを作成・管理する中核部分
    - docker-ce-cli: Dockerを操作するためのコマンドラインツール。docker run や docker ps などのコマンドはこれに含まれる
    - containerd.io: Docker内部でコンテナを実際に実行するランタイム。Dockerだけでなく、Kubernetesなどでも利用されるコンポーネント
    - docker-buildx-plugin: docker buildx コマンドを使うためのプラグイン。マルチプラットフォーム対応のビルド（例：Linux/amd64 と Linux/arm64 両方向けのイメージを1回でビルド）に対応
    - docker-compose-plugin: docker compose コマンドを使うためのプラグイン。複数コンテナをまとめて定義・起動するCompose機能を提供
  - ※ docker-ce（dockered、デーモン） / docker-ce-cli（docker CLI、クライアント） / containerd.io（containerd、コンテナランタイム）を合わせて、Docker Engineという

### 7. sudoなしでDockerを使えるように設定する

- 以下コマンドを実行

  ```bash
  sudo usermod -aG docker $USER
  newgrp docker
  ```

  - 意味：
    - 今ログイン中のユーザを`docker`グループに追加する
    - ログインし直さなくても「いま追加した`docker`グループの権限」を即反映させる

### 8. Ubuntuを起動させたときに自動でDockerも起動されるように設定する

- 以下コマンドを実行

  ```bash
  sudo systemctl enable --now docker
  ```

  - 意味：
    - `enable`で自動起動を設定
    - `--now`で今すぐ起動もする

### 9. Dockerが正しくインストールされたか確認する

- バージョン確認することで、インストールが成功しているか簡単にチェックする

  ```bash
  docker --version
  ```

  出力例：

  ```bash
  Docker version 28.3.3, build 980b856
  ```

- 詳細情報を表示することで、Docker Engine が正常に起動し、ユーザーがアクセスできる状態 かどうかを確認する

  ```bash
  docker info
  ```

  出力例：

  ```bash
  Client: Docker Engine - Community
  Version:    28.3.3
  Context:    default
  Debug Mode: false
  Plugins:
    buildx: Docker Buildx (Docker Inc.)
      Version:  v0.26.1
      Path:     /usr/libexec/docker/cli-plugins/docker-buildx
    compose: Docker Compose (Docker Inc.)
      Version:  v2.39.1
      Path:     /usr/libexec/docker/cli-plugins/docker-compose

  Server:
  Containers: 0
    Running: 0
    Paused: 0
    Stopped: 0
  Images: 0
  Server Version: 28.3.3
  Storage Driver: overlay2
    Backing Filesystem: extfs
    Supports d_type: true
    Using metacopy: false
    Native Overlay Diff: true
    userxattr: false
  Logging Driver: json-file
  Cgroup Driver: systemd
  Cgroup Version: 2
  Plugins:
    Volume: local
    Network: bridge host ipvlan macvlan null overlay
    Log: awslogs fluentd gcplogs gelf journald json-file local splunk syslog
  CDI spec directories:
    /etc/cdi
    /var/run/cdi
  Swarm: inactive
  Runtimes: io.containerd.runc.v2 runc
  Default Runtime: runc
  Init Binary: docker-init
  containerd version: 05044ec0a9a75232cad458027ca83437aae3f4da
  runc version: v1.2.5-0-g59923ef
  init version: de40ad0
  Security Options:
    seccomp
    Profile: builtin
    cgroupns
  Kernel Version: 6.6.87.2-microsoft-standard-WSL2
  Operating System: Ubuntu 24.04.3 LTS
  OSType: linux
  Architecture: x86_64
  CPUs: 8
  Total Memory: 7.621GiB
  Name: DESKTOP-9RA8983
  ID: 4ab3513e-0202-4855-bbae-50777fdb939a
  Docker Root Dir: /var/lib/docker
  Debug Mode: false
  Experimental: false
  Insecure Registries:
    ::1/128
    127.0.0.0/8
  Live Restore Enabled: false
  ```

  - Docker Engineが、WSL2のUbuntu 24.04上に入ったことが読み取れる
  - イメージやコンテナはまだ無い状態

### 10. テストコンテナの実行

  - `--rm`オプションをつけておくことで、コンテナが終了したら自動的にコンテナの削除もしてくれる

  ```bash
  docker run --rm hello-world
  ```

  - 出力例：

  ```bash
  Unable to find image 'hello-world:latest' locally
  latest: Pulling from library/hello-world
  17eec7bbc9d7: Pull complete 
  Digest: sha256:a0dfb02aac212703bfcb339d77d47ec32c8706ff250850ecc0e19c8737b18567
  Status: Downloaded newer image for hello-world:latest

  Hello from Docker!
  This message shows that your installation appears to be working correctly.

  To generate this message, Docker took the following steps:
  1. The Docker client contacted the Docker daemon.
  2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
      (amd64)
  3. The Docker daemon created a new container from that image which runs the
      executable that produces the output you are currently reading.
  4. The Docker daemon streamed that output to the Docker client, which sent it
      to your terminal.

  To try something more ambitious, you can run an Ubuntu container with:
  $ docker run -it ubuntu bash

  Share images, automate workflows, and more with a free Docker ID:
  https://hub.docker.com/

  For more examples and ideas, visit:
  https://docs.docker.com/get-started/
  ```

  - `This message shows that your installation appears to be working correctly.`とあることから、コンテナを作成・実行する一連の流れに成功したことがわかる