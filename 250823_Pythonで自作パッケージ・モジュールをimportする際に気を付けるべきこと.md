「ModuleNotFoundError」になったり、何故かならなかったり、挙動が不安定だった過去を反省し、Pythonで自作パッケージ・モジュールをimportする際に気を付けることを整理してみた。

## 1. __init__.py を入れること

ディレクトリをPythonにパッケージとして認識させるには`__init__.py`が必要である。
これがないと単なるフォルダ扱いとなり、import に失敗する可能性が高い。

```ini
project/
 └── app/
      ├── __init__.py
      ├── main.py
      └── utils.py
```

main.py からは次のように呼び出せる。

```python
from app import utils
```

Python 3.3以降は namespace package により`__init__.py`なしでも動く場合があるが、ツールや実行環境で不安定になるため、実務では必ず置くべきである。

## 2. 相対インポートと絶対インポートを区別すること

### 相対インポート

現在のパッケージを基準に辿る方法である。

```ini
project/
 └── app/
      ├── __init__.py
      ├── main.py
      └── routers
          ├── users.py
          └── items.py
```

```python
from . import utils
from .routers import users
```

上記インポート文で「`.`」が無いと、プロジェクト直下（`project/`）に`routers/`があると解釈されてしまう。「`.`」をつけることで、今のこのファイルがあるパッケージ（今回でいうと`app`）から探してねという意味になる

小規模では便利だが、ネストが深くなると可読性が落ちる。

### 絶対インポート

プロジェクトルートからのフルパスで指定する方法である。

```python
from app import utils
from app.routers import users
```

可読性・保守性が高く、チーム開発では一般的である。ただしプロジェクトルートを PYTHONPATH に通すなど環境設定が必要になる。

## 3. まとめ

- `__init__.py`を必ず置き、パッケージとして認識させること。
- 相対インポートは小規模向き、絶対インポートは実務向きである。
- import エラーの多くは`__init__.py`の不足か、インポート方法の誤りに起因する。