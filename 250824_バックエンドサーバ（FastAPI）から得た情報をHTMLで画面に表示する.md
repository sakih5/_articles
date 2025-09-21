## この記事で取り組んだことの概要

- FastAPI で API サーバを実装したものの、取得した情報を表示させる UI が無かった
- 本記事では、HTML と JavaScript を使用して、API で取得したデータを表示する UI を作成した
- また、動作確認では、仮想環境上に API サーバと Web サーバを立てることで行った

- 補足 具体的な UI 機能:
  - ブラウザで index.html を開くと、FastAPI で作ったエンドポイントから得た JSON を一覧表示できる
  - キーワード検索（`/v1/users?q=...`）や新規作成（`POST /v1/users`）も画面から実行できる

※バックエンドの仕様（エンドポイント、CORS 許可など）は、こちらの記事のサンプルを前提にする

https://zenn.dev/micchi55555/articles/ef956188baae8d

## フォルダ構成

- 環境構築には、`pyenv` + `venv`を使う。Docker は使わない。

```css
frontend-study_fastapi-test/
├── .venv/ /* venv(プロジェクトフォルダを仮想環境とする、Pythonのライブラリが集約されている) */
├── backend/ /* バックエンドのコードを集約 */
│   └── app/
│       ├── routers/
│       └── main.py
├── frontend/ /* フロントエンドのコードを集約 */
│   ├── index.html
│   ├── style.css
│   └── main.js
├── .gitignore /* Git管理しない.venvなどを記載するファイル */
├── .python-version /* pyenv管理で利用するPythonのバージョンを記載するファイル */
└── requirements.txt /* .venvにインストールするライブラリの一覧を記載するファイル */
```

## 事前準備（環境構築）

1. `pyenv`をインストールする。`.venv`を作る

※詳細はこの記事を参照すること

2. `.venv`（仮想環境の中）に入る

```bash
source .venv/bin/activate
```

3. ライブラリを入れる

`requirements.txt`に以下を記載して保存

```txt
fastapi==0.111.0
uvicorn[standard]==0.30.1
pydantic==2.6.4
```

以下コマンドで仮想環境内にインストール

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

## HTML の詳細を解説

### コードのみ全貌

```html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="utf-8" />
    <title>FastAPI → HTML 表示デモ</title>
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <link rel="stylesheet" href="style.css" />
  </head>
  <body>
    <h1>FastAPI のデータを表示</h1>
    <p class="muted">
      バックエンドは <code>/v1/users</code>,
      <code>/v1/items</code> を想定。CORSが許可されている必要があります。
    </p>

    <div class="card">
      <h2>Users 一覧</h2>
      <div class="row">
        <input id="q" placeholder="キーワード検索 (q)" />
        <button id="btnLoadUsers">読み込み</button>
        <!--ボタンの id="btnLoadUsers" をJSが拾ってクリック時にAPIを叩く-->
      </div>
      <ul id="users"></ul>
      <!-- ← ここにJSが<li>を突っ込む -->

      <h3>ユーザー追加 (POST /v1/users)</h3>
      <div class="row">
        <input id="newName" placeholder="name" />
        <button id="btnCreateUser">追加</button>
      </div>
      <p id="usersMsg" class="muted"></p>
    </div>

    <div class="card">
      <h2>Items 一覧</h2>
      <div class="row">
        <input
          id="minPrice"
          type="number"
          step="0.01"
          placeholder="min_price"
        />
        <input
          id="maxPrice"
          type="number"
          step="0.01"
          placeholder="max_price"
        />
        <button id="btnLoadItems">読み込み</button>
        <!--ボタンのid="btnLoadItems" をJSが拾ってクリック時にAPIを叩く-->
      </div>
      <ul id="items"></ul>
      <!--id="users"の空の<ul>が「描画先（マウントポイント）」-->
      <p id="itemsMsg" class="muted"></p>
    </div>

    <script src="./main.js"></script>
    <!--HTMLの一番下で読み込むことで、DOM構築完了後にJSが走る（DOMContentLoaded待ち不要）-->
  </body>
</html>
```

### 各セクション詳細説明

#### 1. 基本設定

```html
<!DOCTYPE html>
<html lang="ja"></html>
```

- `<!DOCTYPE html>`: このページが HTML5 形式 で書かれていることをブラウザに宣言するもの。古い HTML との互換モードを避けて「標準モード」で表示させるために必要
- `<html lang="ja">`: HTML 文書のルート（全体）を囲むタグ。`lang="ja"` → ページの言語は日本語です、とブラウザや検索エンジンに伝える

#### 2. メタ情報の設定

```html
<head>
  <meta charset="utf-8" />
  <title>FastAPI → HTML 表示デモ</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <link rel="stylesheet" href="style.css" />
</head>
```

- `<meta charset="utf-8" />`: ページの文字コードを指定することで、日本語含め文字化けせずに表示できる
- `<title>FastAPI → HTML 表示デモ</title>`: ブラウザのタブに表示させるタイトルを決める
- `<meta name="viewport" content="width=device-width, initial-scale=1" />`:スマホやタブレットで表示したときの拡大縮小を制御するタグ

  - `width=device-width` → 画面幅に合わせてレイアウトする
  - `initial-scale=1` → 初期倍率は 100%

  ← これがないとスマホで表示したときに縮小されて読みにくくなる

- `<link rel="stylesheet" href="style.css" />`: 外部 CSS ファイルを読み込む

#### 3. ページタイトルと説明

```html
<h1>FastAPI のデータを表示</h1>
<p class="muted">
  バックエンドは <code>/v1/users</code>, <code>/v1/items</code> を想定。
  CORSが許可されている必要があります。
</p>
```

- `<h1>` → ページの見出し（大きなタイトル）
- `<p class="muted">` → 補足説明。「muted」クラスで文字色をグレーにしてる
- `<code>` → コード風のフォントで表示。エンドポイント名を強調

#### 4. ユーザ一覧表示

```html
<div class="card">
  <h2>Users 一覧</h2>
  <div class="row">
    <input id="q" placeholder="キーワード検索 (q)" />
    <button id="btnLoadUsers">読み込み</button>
  </div>
  <ul id="users"></ul>
</div>
```

- `div.card` → 枠付きのカード風レイアウト
- `<input id="q">` → ユーザー検索用のテキストボックス
- `<button id="btnLoadUsers">` → 読み込みボタン（JavaScript が拾って API 呼び出しを実行）
- `<ul id="users">` → 結果を表示するリスト。最初は空で、JS が <li> を追加していく

#### 5. ユーザ追加フォーム

```html
<h3>ユーザー追加 (POST /v1/users)</h3>
<div class="row">
  <input id="newName" placeholder="name" />
  <button id="btnCreateUser">追加</button>
</div>
<p id="usersMsg" class="muted"></p>
```

- `<input id="newName">` → 新しいユーザー名を入力する欄
- `<button id="btnCreateUser">` → クリックで POST 送信
- `<p id="usersMsg">` → メッセージ表示用（「追加しました」「エラー: ...」などがここに出る）

#### 6. Items 一覧カード

```html
<div class="card">
  <h2>Items 一覧</h2>
  <div class="row">
    <input id="minPrice" type="number" step="0.01" placeholder="min_price" />
    <input id="maxPrice" type="number" step="0.01" placeholder="max_price" />
    <button id="btnLoadItems">読み込み</button>
  </div>
  <ul id="items"></ul>
  <p id="itemsMsg" class="muted"></p>
</div>
```

- `minPrice` / `maxPrice` → 数値入力欄（`type="number"`で数字専用）
- `step="0.01"` → 小数点第 2 位まで入力可能
- `<button id="btnLoadItems">` → API `/v1/items` を呼んで一覧を取得
- `<ul id="items">` → アイテム一覧を JS がここに挿入
- `<p id="itemsMsg">` → 件数やエラーメッセージを表示

#### 7. JavaScript の読み込み

```html
<script src="./main.js"></script>
```

- この HTML で使っている動作ロジックは全部 `main.js` に書かれている
- 一番下に書く理由
  → DOM 構築（HTML の要素が全部作られる）のを待ってから JS を実行できる
  →「まだ <ul id="users"> が無いのに JS が動いてしまう」といった事故を防げる

## CSS の詳細を解説

### コード全貌

```css
body {
  font-family: system-ui, sans-serif; /* フォント指定（OS標準フォント） */
  margin: 24px; /* ページの余白 */
  line-height: 1.6; /* 行間 */
}

.card {
  border: 1px solid #ddd; /* 薄い枠線 */
  border-radius: 12px; /* 角丸 */
  padding: 16px; /* 内側の余白 */
  margin: 12px 0; /* 上下の余白 */
}

.row {
  display: flex; /* 横並びレイアウト */
  gap: 8px; /* 要素間の間隔 */
  flex-wrap: wrap; /* はみ出したら折り返す */
  align-items: center; /* 縦方向を中央に揃える */
}

input,
button {
  padding: 8px 12px; /* 内側の余白（クリックしやすく） */
}

ul {
  padding-left: 20px; /* 箇条書きのインデント */
}

.muted {
  color: #666; /* 灰色っぽい文字 */
  font-size: 0.9em; /* 少し小さい文字 */
}

.error {
  color: #b00020; /* エラー用の赤色 */
}
```

## JavaScript の詳細を解説

- HTML で作った 3 つのボタンを押したときの動作を Javascript で定義する
- 3 種類のボタンを用意
  - loadUsers ボタン
  - CreateUser ボタン
  - loadItems ボタン
- フォームに入力された値をもとに、HTTP リクエストに変換する関数`qs`を事前に定義しておく

### コードのみ全貌

```javascript
const API_BASE = 'http://localhost:8000';

const qs = (obj) =>
  Object.entries(obj)
    .filter(([, v]) => v !== undefined && v !== null && v !== '')
    .map(([k, v]) => `${encodeURIComponent(k)}=${encodeURIComponent(v)}`)
    .join('&');

async function loadUsers() {
  const q = document.getElementById('q').value.trim();
  const url = `${API_BASE}/v1/users/?${qs({ q, limit: 50, offset: 0 })}`;
  const ul = document.getElementById('users');
  const msg = document.getElementById('usersMsg');
  ul.innerHTML = '';
  msg.textContent = '読み込み中...';

  try {
    const res = await fetch(url);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const data = await res.json();
    if (!Array.isArray(data)) throw new Error('Unexpected response');
    data.forEach((u) => {
      const li = document.createElement('li');
      li.textContent = `#${u.id} ${u.name}`;
      ul.appendChild(li);
    });
    msg.textContent = `${data.length}件表示`;
  } catch (e) {
    msg.textContent = `エラー: ${e.message}`;
    msg.classList.add('error');
  }
}

async function createUser() {
  const name = document.getElementById('newName').value.trim();
  const msg = document.getElementById('usersMsg');
  msg.textContent = '送信中...';
  try {
    const res = await fetch(`${API_BASE}/v1/users/`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ name }),
    });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    await loadUsers(); // ユーザーを追加した直後に、一覧を再取得して画面を最新化
    document.getElementById('newName').value = '';
    msg.textContent = '追加しました';
  } catch (e) {
    msg.textContent = `エラー: ${e.message}`;
    msg.classList.add('error');
  }
}

async function loadItems() {
  const min = document.getElementById('minPrice').value;
  const max = document.getElementById('maxPrice').value;
  const url = `${API_BASE}/v1/items/?${qs({
    min_price: min || undefined,
    max_price: max || undefined,
  })}`;

  const ul = document.getElementById('items');
  const msg = document.getElementById('itemsMsg');

  ul.innerHTML = '';
  msg.textContent = '読み込み中...';

  try {
    const res = await fetch(url);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const data = await res.json();

    data.forEach((it) => {
      const li = document.createElement('li');
      li.textContent = `#${it.id} ${it.name} (¥${it.price})`;
      ul.appendChild(li);
    });

    msg.textContent = `${data.length}件表示`;
  } catch (e) {
    msg.textContent = `エラー: ${e.message}`;
    msg.classList.add('error');
  }
}

// ボタンをイベントに登録
document.getElementById('btnLoadUsers').addEventListener('click', loadUsers);
document.getElementById('btnCreateUser').addEventListener('click', createUser);
document.getElementById('btnLoadItems').addEventListener('click', loadItems);

// 初期表示
loadUsers();
loadItems();
```

#### 各セクション詳細説明

#### 1. 接続先（バックエンドサーバー）の指定

```js
const API_BASE = 'http://localhost:8000';
```

- `API_BASE`: バックエンドのベース URL。WSL や Docker の設定に合わせて変更する
  - `"http://localhost:8000"`: FastAPI の Uvicorn が:8000 で待受している場合はこう書く
  - `"/api"`: `API_BASE`に代入する値をこれにすると、Docker の nginx 経由になる

#### 2. オブジェクト（連想配列）を HTTP リクエスト用のクエリ文字列に変換する関数の定義

```js
const qs = (obj) =>
  Object.entries(obj)
    .filter(([, v]) => v !== undefined && v !== null && v !== '')
    .map(([k, v]) => `${encodeURIComponent(k)}=${encodeURIComponent(v)}`)
    .join('&');
```

- この関数の使用例:

  ```js
  qs({ q: 'ali', limit: 50 });
  // => "q=ali&limit=50"
  ```

- アロー関数式で定義されている。`=>`の手前にある`()`に書かれたものが引数、`=>`の先にあるものが返り値

  - 引数: 辞書（Object 型）。例: `{q: "ali", limit: 50}`
  - 返り値: 文字列（String 型）。例: `"q=ali&limit=50"`

- `Object.entries(obj)`:`obj`を`[key, value]`の配列に変換

  - 例:`{q: "ali", limit: 50}`→`[["q", "ali"], ["limit", 50]]`

- `.filter(([, v]) => v !== undefined && v !== null && v !== '')`: 値が`undefined`/`null`/空文字のものを除外

  - 例: `[["q", "ali"], ["limit", 50], ["sort", ""]]` → `[["q", "ali"], ["limit", 50]]`

- `` .map(([k, v]) => `${encodeURIComponent(k)}=${encodeURIComponent(v)}`) ``: 各`[key, value]`を`key=value`形式に変換。`encodeURIComponent`を使って URL エンコード（スペース →`%20`など）

  - 例: `[["q", "ali"], ["limit", 50]]` → `["q=ali", "limit=50"]`

- `.join('&')`: 配列を & で結合して、最終的なクエリ文字列にする

  - 例: `["q=ali", "limit=50"]` → `"q=ali&limit=50"`

- 補足: `encodeURIComponent`の役割

  - クエリ文字列は URL の一部なので、特殊文字（スペース、記号、日本語など）をそのまま入れると壊れる可能性がある
  - encodeURIComponent によって安全に変換される

  ```js
  qs({ keyword: 'C++ 入門' });
  // "keyword=C%2B%2B%20%E5%85%A5%E9%96%80"
  ```

#### 3. ユーザ一覧のデータを取得＆HTML に表示する用に加工する関数の定義

```js
async function loadUsers() {
  const q = document.getElementById('q').value.trim();
  const url = `${API_BASE}/v1/users/?${qs({ q, limit: 50, offset: 0 })}`;
  const ul = document.getElementById('users');
  const msg = document.getElementById('usersMsg');
  ul.innerHTML = '';
  msg.textContent = '読み込み中...';

  try {
    const res = await fetch(url);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const data = await res.json();
    if (!Array.isArray(data)) throw new Error('Unexpected response');
    data.forEach((u) => {
      const li = document.createElement('li');
      li.textContent = `#${u.id} ${u.name}`;
      ul.appendChild(li);
    });
    msg.textContent = `${data.length}件表示`;
  } catch (e) {
    msg.textContent = `エラー: ${e.message}`;
    msg.classList.add('error');
  }
}
```

- `async function`: 非同期処理を行う関数の定義

  - `async`: この関数は「非同期処理（Promise を返す関数）」ですと宣言するキーワード
  - `await`: Promise の「結果が返るまで待つ」構文
  - ポイント:

    1. async function の中だけで await が使える
    2. await を書いた行は、Promise が解決されるまで処理が一時停止する
    3. その結果、次の行の処理は「結果を受け取ってから」実行される
    4. これにより .then(...) をチェーンで書かずに、同期処理っぽく書ける

- `const q = document.getElementById("q").value.trim();`:

  - `document`: ブラウザが表示しているページ全体（HTML 文書）を表すオブジェクト
  - `getElementById`: 「この HTML の中から、指定した`id`を持つ要素を 1 つ取り出す」メソッド

    - HTML のルール上、id 属性は一意であるべきで、同じ id を 2 つ以上つけないでねという前提があるが、
    - もし誤って同じ id を 2 つ以上つけてしまった場合、最初に見つかった 1 つ目が返ってきて 2 つ目以降は無視される
    - 補足 セレクタの例:

      - `getElementById`: id が指定値のものを 1 つだけ取得できる
      - `getElementsByClassName`: class が指定値のものを複数まとめて取得できる
      - `querySelectorAll`: CSS セレクタが指定値のものを複数まとめて取得できる ← これがもっともよく使われる

  - `.value`: フォーム要素（input, textarea など）の「入力値」を取得／設定するプロパティ
  - `.trim()`: 文字列の前後の空白（スペース, 改行, タブ）を削除するメソッド

- `const ul = document.getElementById("users"); const msg = document.getElementById("usersMsg");`: ul と msg は、「値」ではなく「画面の部品要素（`<ul>`や`<p>`**そのもの**）」を参照するため、値ではなく DOM 要素をそのまま変数に代入
- `ul.innerHTML = "";`: `innerHTML`は、要素の「中身の HTML 文字列」の意

  - `""`を代入している → 中身を空っぽにして、リストを初期化している
  - `ul`は**定数**（参照先の要素を指す）だけど、その中身のプロパティを変更するのは OK

- `msg.textContent = "読み込み中...";`: `textContent` = 要素の「テキスト部分」を表すプロパティ

  - `"読み込み中..."`を代入 → その要素の表示テキストが置き換わる

- `try {...} catch (e) {...}`: `try`に続く`{}`の中には正常系の処理を、`catch (e)`に続く`{}`の中には「正常系の処理のどこかでエラーが出たときの処理」を書く

  - `const res = await fetch(url);`: HTTP リクエストを送り、サーバからレスポンスが返ってくるまで一時停止する
  - `` if (!res.ok) throw new Error(`HTTP ${res.status}`); ``:

    - `!`は否定を表す演算子。`!res.ok`は成功じゃないとき
    - `throw` = 「例外（エラー）を投げる」構文
    - `new Error("メッセージ")` = エラーオブジェクトを作る
    - 合わせると「エラーを発生させて処理を中断し、catch に飛ばす」という動きになる

  - `const data = await res.json();`:

    - レスポンスを JSON に変換
    - `res.json()`はレスポンスの本文を JSON として読み込む（これも非同期）
    - `data`には JS のオブジェクトが入る

  - `if (!Array.isArray(data)) throw new Error("Unexpected response");`:

    - データが配列か確認
    - 想定通り「配列」じゃなければエラー扱い
    - これで「API の返却が壊れていた」場合にも安全に処理できる

  - `` data.forEach((u) => {const li = document.createElement("li"); li.textContent = `#${u.id} ${u.name}`; ul.appendChild(li); }); ``:

    - DOM にリストを追加する
    - `forEach`でデータの配列を 1 件ずつ処理していく
    - `document.createElement("li")`→ 新しい`<li>`要素を作成
    - `li.textContent = ...`で文字列を入れる
    - `ul.appendChild(li)`で`<ul>`に追加

  - `` msg.textContent = `${data.length}件表示`; ``:

    - `msg`はメッセージ表示用の要素
    - 配列の長さを数えて「○ 件表示」と表示

  - `` catch (e) { msg.textContent = `エラー: ${e.message}`;  msg.classList.add("error"); } ``:

    - エラーが起きたら（ネットワーク失敗、JSON 壊れてるなど）ここに来る
    - メッセージに`エラー: ...`を出す
    - `.classList.add("error")`で CSS のクラス`error`を追加（赤字にするなど）

#### 4. ユーザを追加 & HTML に表示する内容を更新する関数の定義

```js
async function createUser() {
  const name = document.getElementById('newName').value.trim();
  const msg = document.getElementById('usersMsg');
  msg.textContent = '送信中...';
  try {
    const res = await fetch(`${API_BASE}/v1/users/`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ name }),
    });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    await loadUsers(); // ユーザーを追加した直後に、一覧を再取得して画面を最新化
    document.getElementById('newName').value = '';
    msg.textContent = '追加しました';
  } catch (e) {
    msg.textContent = `エラー: ${e.message}`;
    msg.classList.add('error');
  }
}
```

- `async function`: 非同期処理を行う関数（`await`が中で使える）
- `const name = document.getElementById('newName').value.trim();`: 入力値を取り出す

  - `id="newName"`の`<input>`から入力値を取得
  - `.trim()`で前後の空白を削除

- `const msg = document.getElementById('usersMsg');`: 現在表示されているメッセージの DOM をまるっと取得する
- `msg.textContent = '送信中...';`: 表示を「送信中...」に更新して、処理中であることをユーザーに知らせる
- `` const res = await fetch(`${API_BASE}/v1/users/`, { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ name }), }); ``: サーバに POST リクエスト

  - `fetch`に第 2 引数を渡すと詳細なリクエスト設定ができる
  - `method: "POST"` → 新規作成リクエスト
  - `headers: { "Content-Type": "application/json" }` → JSON を送るよ、という宣言
  - `body: JSON.stringify({ name })` → { "name": "Alice" } のような JSON 文字列に変換して送信

- `` if (!res.ok) throw new Error(`HTTP ${res.status}`); ``: ステータス確認。成功（200 系）以外ならエラーを投げて catch に飛ばす
- `await loadUsers();`: ユーザーを追加した直後に、一覧を再取得して画面を最新化
- `document.getElementById("newName").value = "";`: 入力欄を空に戻す
- `msg.textContent = "追加しました";`: メッセージを「追加しました」に更新
- `` catch (e) { msg.textContent = `エラー: ${e.message}`;  msg.classList.add("error"); } ``:

  - エラーが起きたら（ネットワーク失敗、JSON 壊れてるなど）ここに来る
  - メッセージに`エラー: ...`を出す
  - `.classList.add("error")`で CSS のクラス`error`を追加（赤字にするなど）

#### 5. 商品一覧のデータを取得＆HTML に表示する用に加工する関数の定義

```js
async function loadItems() {
  const min = document.getElementById('minPrice').value;
  const max = document.getElementById('maxPrice').value;
  const url = `${API_BASE}/v1/items/?${qs({
    min_price: min || undefined,
    max_price: max || undefined,
  })}`;

  const ul = document.getElementById('items');
  const msg = document.getElementById('itemsMsg');

  ul.innerHTML = '';
  msg.textContent = '読み込み中...';

  try {
    const res = await fetch(url);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const data = await res.json();

    data.forEach((it) => {
      const li = document.createElement('li');
      li.textContent = `#${it.id} ${it.name} (¥${it.price})`;
      ul.appendChild(li);
    });

    msg.textContent = `${data.length}件表示`;
  } catch (e) {
    msg.textContent = `エラー: ${e.message}`;
    msg.classList.add('error');
  }
}
```

- `async function`: 非同期処理を行う関数（`await`が中で使える）
- `const min = document.getElementById('minPrice').value; const max = document.getElementById('maxPrice').value;`: 入力値を取り出す

  - `<input id="minPrice">` と `<input id="maxPrice">` の入力値を取得
  - まだ文字列（例: `"100"`, `"200"`）のまま

- `` const url = `${API_BASE}/v1/items/?${qs({ min_price: min || undefined,  max_price: max || undefined, })}`; ``: クエリ文字列を組み立てる

  - `qs({ ... })` → オブジェクトを`key=value&...`のクエリ文字列に変換する自作関数
  - `||`は論理 OR 演算子といって、もし左が使えないときは右を使うの意

    - `min || undefined`: 入力が空なら`undefined`にする（`qs`側で除外されるのでクエリに含めない）
    - 例：
    - 入力が`min=100`, `max=200` → `/v1/items/?min_price=100&max_price=200`
    - 入力が空 → `/v1/items/`

- `  const ul = document.getElementById('items'); const msg = document.getElementById('itemsMsg');`: 描画先とメッセージ要素を取得

  - `<ul id="items">` … アイテムリストを並べる場所 `<p id="itemsMsg">` … 状態表示メッセージ（「読み込み中…」「エラー…」など）

- `ul.innerHTML = ""; msg.textContent = "読み込み中...";`: 初期化

  - リストの中身を一旦空にする
  - メッセージを「読み込み中...」に変えてユーザーに処理中と知らせる

- `const res = await fetch(url);`: サーバにリクエスト（API にアクセス）
- `` if (!res.ok) throw new Error(`HTTP ${res.status}`); ``: ステータスが 200 系でなければエラーにする
- `const data = await res.json();`: レスポンス本文を JSON に変換 → data が配列になる
- `` data.forEach((it) => { const li = document.createElement("li"); li.textContent = `#${it.id} ${it.name} (¥${it.price})`; ul.appendChild(li); }); ``: data を DOM に変換

  - `forEach`: データを 1 つずつ取り出して、
  - `const li = document.createElement("li");`: `<li>`要素を作成し、
  - `` li.textContent = `#${it.id} ${it.name} (¥${it.price})`; ``: id, name, price を文字列として入れる
  - `ul.appendChild(li);`: `<ul>`に追加していく → ブラウザにリスト表示される

- `` msg.textContent = `${data.length}件表示`; ``: 件数表示
- `` catch (e) { msg.textContent = `エラー: ${e.message}`;  msg.classList.add("error"); } ``:

  - エラーが起きたら（ネットワーク失敗、JSON 壊れてるなど）ここに来る
  - メッセージに`エラー: ...`を出す
  - `.classList.add("error")`で CSS のクラス`error`を追加（赤字にするなど）

#### 6. ボタンにイベントを登録

```js
document.getElementById('btnLoadUsers').addEventListener('click', loadUsers);
document.getElementById('btnCreateUser').addEventListener('click', createUser);
document.getElementById('btnLoadItems').addEventListener('click', loadItems);
```

- `document.getElementById("...")` → HTML の中から id 属性で要素を探す
- `.addEventListener("click", 関数)` → 「この要素がクリックされたときに、この関数を実行する」という設定

  - 「ユーザー読み込み」ボタンがクリックされたら → `loadUsers()` 実行
  - 「ユーザー追加」ボタンがクリックされたら → `createUser()` 実行
  - 「アイテム読み込み」ボタンがクリックされたら → `loadItems()` 実行

- `.addEventListener("click", 関数)`で関数のところを`loadUsers`とする（`loadUsers()`としない）理由
  - → `()`がつくと「その場で関数を実行して、その戻り値を渡す」という意味になる。もし戻り値が`undefined`なら → `addEventListener("click", undefined)`になってしまって、イベントは登録されない・つまり「ページ読み込み時に即実行」されてしまって、ボタンを押しても反応しなくなる
  - ← `()`が無いと loadUsers という「関数オブジェクトそのもの」を渡している・ボタンがクリックされたときに あとで呼び出してね という登録になる

#### 7. ページ表示直後にデータを読み込む（初期表示）

```js
loadUsers();
loadItems();
```

- ページを開いた瞬間に 自動でユーザー一覧とアイテム一覧を表示させる
- これをしないと最初は空っぽの画面になってしまう

## 動作確認

1. バックエンド（FastAPI）を起動する

- （ターミナルを新たに開くなどして）セッション内でまだ仮想環境に入っていないようであれば、入る

  ※ 仮想環境に入っているかどうかは、ターミナルの行頭を見ればわかる。`(.venv)`を仮想環境名が書かれていれば、仮想環境に入れている

  ```bash
  cd /home/sakih/projects/frontend-study_fastapi-test
  source .venv/bin/activate
  ```

- 以下コマンドを 1 行ずつ実行してバックエンドサーバを立ち上げる

  ```bash
  cd /home/sakih/projects/frontend-study_fastapi-test/backend
  uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
  ```

  - 意味:

    - `uvicorn`: Uvicorn（ASGI サーバ） を使って FastAPI アプリを起動するコマンド
    - `app.main`: `app/`ディレクトリの`main.py`ファイルを指している。Python のモジュール指定の書き方（`app/main.py`）
    - `:app`: `main.py`の中にある FastAPI インスタンス（変数） の名前が`app`だよ、という意
    - `--reload`: ソースコードを変更したら、自動でサーバを再起動してくれるオプション。開発中に便利（本番環境では普通オフにする）
    - `--host 0.0.0.0`: サーバが待ち受けるアドレス。`0.0.0.0`は「どの IP からのアクセスでも受け付ける」という意味。ローカルだけなら`127.0.0.1`でもいいけど、他の PC や仮想環境からアクセスしたいときは`0.0.0.0`が必要
    - `--port 8000`: サーバが待ち受けるポート番号。FastAPI のデフォルトは 8000。もし既に何かが 8000 を使っていたら --port 8080 などに変更できる

    → 要は、「`app/main.py`にある`app`を、Uvicorn でポート 8000 に載せて、どこからでもアクセスできるようにし、コード変更時は勝手に再起動してね！」という意

- http://localhost:8000/docs を開いて API サーバが立ち上がったことを確認

2. フロントを配信

- 前のステップでバックエンドサーバを立ち上げ中で、新たにコマンドを打てなくなっているため、新しいシェルを開く。**VSCode 上のターミナルの画面の右上の「+」ボタンを押す**
- 新たに開いたシェルに以下コマンドを 1 行ずつ実行してフロントエンドサーバを立ち上げる

```bash
cd ~/projects/frontend-study_fastapi-test/frontend
python -m http.server 5173
```

- 意味: **今いるフォルダの中身を Web サーバで配信**するという意

  - `python`: Python インタプリタを実行する
  - `-m http.server`:
    - `-m`は「モジュールをスクリプトとして実行する」というオプション
    - `http.server`は Python 標準ライブラリのモジュールで、静的ファイルを配信できるサーバ機能を持っている
    - これにより「現在のディレクトリのファイルを Web で公開する」サーバが立ち上がる
  - `5173`: ポート番号の指定。デフォルトでは 8000 だが、ここで 5173 を明示的に指定
    - 5173 は Vite（React や Vue でよく使われる開発サーバ）がデフォルトで使う番号なので、それに合わせた

- http://localhost:5173/ にアクセスしてフロントエンドサーバが立ち上がったことを確認
  - Web サーバには「ルート URL（例: /）にアクセスされたら、特定のファイルを自動で返す」ルールがある
    - 多くの場合、それは`index.html`
    - これは「そのフォルダのトップページ」という意味で使われる

## 感想・気づき

- UI のボタンを押せば API が直接叩けると思っていたのが、ちょっと間違っていたことに気づいた
  - ボタン自体は HTML で置けるが、ボタンを押した後の画面変化とデータ取得（through API）は JavaScript が行っているんだとわかった
  - UI のボタンを押せば API が直接叩けるのはそうなのだが、HTML と API サーバの間に JavaScript があるというのが、発見だった
- JavaScript は何のためにある？と思っていたのだが、実際にコードを書いて作ってみることで理解できるようになった
  - JavaScript の役割:
    1. ボタンを押した後、HTML のどこのタグの値をどう変えるか定義する
    2. 入力された値をクエリ文字列に変換し、HTTP リクエストを作成する
    3. API サーバから受け取った JSON を、HTML に変換する
- なんで HTML は indx.html という名前にする？と思っていたのが、理由が分かってスッキリした
