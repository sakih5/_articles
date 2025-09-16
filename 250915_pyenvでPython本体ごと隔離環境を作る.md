# pyenv で Python 本体ごと隔離環境を作る

pyenvとは、チームで環境を揃えるための**Pythonバージョン管理ツール**

## 前提: この記事でやること

- **WSL2のUbuntu-24.04**上にpyenvをインストールする
- ライブラリはプロジェクトフォルダ内に設定した**venv**上にインストールする

    → pyenv + venv で「Pythonバージョン + ライブラリ環境」を完全に再現する

## 背景: pyenvを使用する目的

- Pythonの**バージョン**を指定して開発環境を構築するため
- OS（Windows PCやWSL上）に直接インストールされたPythonがあるPC上でも、pyenvがあれば**別のバージョンのPythonを利用**することができる
- **pyenv管理配下**だと、異なるバージョンのPythonを**プロジェクト単位・ユーザ単位**で切り替えて利用することができる

### 補足: Pythonのバージョンを指定して開発することのメリット

- 本番サーバーやCIで利用するPythonバージョンに合わせてローカル環境を構築できるため、**ローカルでは動いたのにデプロイ後にエラー**といったリスクを下げられる

## 手順

### 1. pyenvをインストールするために必要なパッケージを先にインストールしておく

- まず手元のパッケージリストを更新する。`apt`はUbuntu/Debian系のパッケージ管理ツール

```bash
sudo apt update
```

- パッケージをインストール。各パッケージの役割は下表を参照

| パッケージ名                            | 役割・必要な理由                                                                                             |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| **build-essential**                    | `gcc`, `g++`, `make` などをまとめたパッケージ。C言語で書かれたPython本体や拡張モジュールをコンパイルするために必須。これが無いと「コンパイラが無い」とエラーになる。     |
| **libssl-dev**                         | OpenSSL 開発用ヘッダ。Pythonの `ssl` モジュールに必要。HTTPS通信、証明書検証、pip でのライブラリインストールなど「安全な通信」ができなくなるので必須。           |
| **zlib1g-dev**                         | zlib 圧縮ライブラリ。Python の `zlib` モジュールに対応し、`gzip` 圧縮ファイルの読み書きに使われる。`pip install` 時の圧縮解凍にも利用。             |
| **libbz2-dev**                         | bzip2 圧縮ライブラリ。`.bz2` 形式のファイルを扱うために必要。`tar.bz2` 形式のアーカイブを解凍するモジュールに依存。                                |
| **libreadline-dev**                    | GNU Readline ライブラリ。Python 対話モード(REPL)で矢印キーで履歴を辿れたり、Ctrl+Rで検索できるのはこれのおかげ。無いとREPLが不便になる。               |
| **libsqlite3-dev**                     | SQLite データベースライブラリ。Python 標準の `sqlite3` モジュールをビルドするために必要。これが無いとSQLiteが使えなくなる。                        |
| **curl**                        | ファイルダウンロードツール。pyenvがPython公式サイトからソースコードを取得する際に使う。                                |
| **libncurses-dev** | ターミナル制御ライブラリ。Python の `curses` モジュールを有効にするために必要。テキストベースUIアプリ（例: htop, vim）の基盤になる。                    |
| **xz-utils**                           | `.xz` 圧縮形式を展開するためのツール。Pythonの公式ソース配布は `.tar.xz` なので必須。これが無いと `pyenv install` でソース展開に失敗する。            |
| **tk-dev**                             | GUI ライブラリ Tkinter のヘッダ。Python の `tkinter` モジュールを有効にするために必要。GUIアプリを作らないなら無くても良いが、標準で含めたいなら必要。         |
| **libffi-dev**                         | Foreign Function Interface。Python の `ctypes` や `cffi` モジュールが外部ライブラリを呼び出すのに必要。科学計算や暗号ライブラリが内部で依存している。 |
| **liblzma-dev**                        | LZMA 圧縮ライブラリ。`.xz` や `.lzma` ファイルを扱う `lzma` モジュールに必要。圧縮形式が扱えないと一部のpipパッケージ展開で不便になる。                  |
| **git**                                | pyenv 本体やプラグインを GitHub から clone するために必要。pyenv を使う上で必須。                                               |

```bash
sudo apt install -y \
  build-essential \
  libssl-dev \
  zlib1g-dev \
  libbz2-dev \
  libreadline-dev \
  libsqlite3-dev \
  curl \
  libncurses-dev \
  xz-utils \
  tk-dev \
  libffi-dev \
  liblzma-dev \
  git
```

### 2. pyenvをインストールする

```bash
git clone https://github.com/pyenv/pyenv.git ~/.pyenv
```

### 3. pyenvをシェルから使えるように設定する

```bash
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
```

- 1行目コマンド説明: ホームディレクトリ（`~`）直下の`.pyenv`ディレクトリのパス（`~/.pyenv`）を、`PYENV_ROOT`という変数名で環境変数として命名する
- 2行目コマンド説明: PATHに`pyenv`の実行ファイルディレクトリ（`~/.pyenv/bin`）を追加する。先頭に入れることで、システム標準の`python`や`pip`よりも`pyenv`が優先して使われるようにする

    → パスを通すことで、シェルで`pyenv`コマンドが使えるようになる

- 3行目コマンド説明: `pyenv init -`で初期化用のシェルスクリプトを文字列として出力。`"$( ... )"`でその出力を文字列として取り込み、`eval`で取り込んだ文字列を実際のシェルコマンドとして実行する

    `pyenv init -`のみでの実行結果は以下

    ```bash
    $ pyenv init -
    PATH="$(bash --norc -ec 'IFS=:; paths=($PATH); 
    for i in ${!paths[@]}; do 
    if [[ ${paths[i]} == "''/home/user/.pyenv/shims''" ]]; then unset '\''paths[i]'\''; 
    fi; done; 
    echo "${paths[*]}"')"
    export PATH="/home/user/.pyenv/shims:${PATH}"
    export PYENV_SHELL=bash
    source '/home/user/.pyenv/completions/pyenv.bash'
    command pyenv rehash 2>/dev/null
    pyenv() {
    local command=${1:-}
    [ "$#" -gt 0 ] && shift
    case "$command" in
    rehash|shell)
        eval "$(pyenv "sh-$command" "$@")"
        ;;
    *)
        command pyenv "$command" "$@"
        ;;
    esac
    }
    ```

    → shims経由でPythonが実行されるようになる

```bash
cp -a ~/.bashrc ~/.bashrc.bak.$(date +%Y%m%d-%H%M%S) # バックアップ作成

grep -n "pyenv init" ~/.bashrc || true # 上記3行のコマンドを既に書いていないか確認
grep -n "PYENV_ROOT" ~/.bashrc || true # 上記3行のコマンドを既に書いていないか確認

# ~/.bashrcに書き込み
cat <<'EOF' >> ~/.bashrc

# >>> pyenv initialize >>>
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
# <<< pyenv initialize <<<
EOF

# 書き込んだ内容を実行して設定を反映（既に手動で設定済みの内容ではあるが、~/.bashrcへの書き方に問題がないことを確認する意図で実行する）
source ~/.bashrc
```

- 上記3行をWSL2を起動して新しいシェルを開く（bashが起動する）たびに毎回実行されるよう`~/.bashrc`に書き込みする（自動でpyenvが使える状態になる）

#### 保留: 上記やり方では、bash起動後にしかpyenvが使えない。ログイン直後にpyenvが使えるようにするためには

- `~/.profile`の末尾に`eval "$(pyenv init --path)"`と追記しておく
- ただし、Debian系では、ログインの際に`~/.profile`を参照しないとLPIC1で学んだはず。`~/.profile`を作ることの意味があるのか疑問。

    → とりあえず、これはやらなかった。

### 4. Pythonをインストールする

```bash
pyenv install -l | grep 3.1
pyenv install 3.11.13
pyenv install 3.12.11
```

- 1行目コマンド説明: `pyenv install -l`でpyenvでインストール可能なPythonバージョンの一覧を表示。` | grep 3.1`でバージョン名のうち、「3.1」という文字列を含むものに絞り込んで表示
- 2,3行目コマンド説明: 実際にあったバージョンのPythonをインストールする（今回は`3.11.13`と`3.12.11`をインストールした）

### 5. バージョンの設定・切り替え

- グローバル（ユーザー全体で使うデフォルト）

```bash
pyenv global 3.11.13
```

- プロジェクトごと（ディレクトリに`.python-version`ができる）

```bash
cd ~/projects/myapp
pyenv local 3.11.13
```

### 6. venvの作成

- pyenvで選んだPython上に仮想環境を作る
- 環境によっては、`ensurepip`が入っておらず失敗することがある。
- → `python3.x-venv`パッケージが必要という意
- → もしvenv作成でエラーが出たら`sudo apt install python3.x-venv`を追加

```bash
cd ~/projects/myapp # プロジェクトフォルダに移動
python -m venv .venv # 仮想環境（名前.venv）を作成
source .venv/bin/activate # 仮想環境に入る
```

### 7. インストール＆設定できているか確認

1. pyenv本体のバージョンを表示する

```bash
user@host:~$ pyenv --version
pyenv 2.6.7-16-g857806e6
```

2. pyenv でインストール済みのPython一覧を表示

- `*`がついているものが「今選ばれている」バージョン

```bash
user@host:~$ pyenv versions
  system
* 3.11.13 (set by /home/user/projects/frontend-study_fastapi-test/.python-version)
  3.12.11
```

3. 実際に呼び出される`python`のバージョンを表示する

- pyenvの設定が効いているかどうかを確認する基本のコマンド
- pyenvで指定したバージョンが出れば成功

```bash
user@host:~$ python --version
Python 3.11.13
```

4. シェルが`python`コマンドを探す順番を表示する

- pyenv の「shim経由」で実際のPythonが選ばれていればOK

```bash
user@host:~$ type -a python
python is /home/sakih/.pyenv/shims/python
```

5. 参考）pyenvのプラグイン`pyenv-doctor`を使った確認

- `~/.pyenv/plugins/`ディレクトリがあることを確認した上で、`pyenv-doctor`をインストールする

```bash
git clone https://github.com/pyenv/pyenv-doctor.git "$(pyenv root)/plugins/pyenv-doctor"
```

- インストールできたら実際に使ってみる。`Congratulations! You are ready to build pythons!`と出ればOK

```bash
user@host:~/projects/api-test$ pyenv doctor
Cloning /home/user/.pyenv/plugins/pyenv-doctor/bin/.....
Installing python-pyenv-doctor...
Installed python-pyenv-doctor to /tmp/pyenv-doctor.20250916091053.8770/prefix
Congratulations! You are ready to build pythons!
```
