FastAPIを使うと、自動でSwagger UIが生成される。これにより、エンドポイントの仕様確認や動作テストを直感的に行うことができる

この記事では、5ステップに分けてSwagger UIの見方を整理する

## 1. Swagger UIの基本情報表示

![](imgs\FastAPIのSwaggerUI画面説明1.png)

- **タイトル / バージョン / 説明**: `main.py`で指定した`title, version, description`が反映される
- `/openapi.json`: API仕様をJSON形式で返すURL。Swagger UIはここを裏で読み込んでいる
- `OAS 3.1`:OpenAPI Specificationのバージョンを示している（REST APIの標準規格）

## 2. エンドポイントの一覧と詳細

![](imgs\FastAPIのSwaggerUI画面説明2.png)

- **タグごとのグルーピング**: `APIRouter(tags=["users"])`のように設定すると、users タグでまとめられる
- **HTTPメソッドとパス**: `GET /v1/users/`のように、メソッドとパスが表示される
- **関数名・Docstring**: 関数名がキャメルケース風に整形されて表示され、Docstringの内容が説明に反映される
- **リクエスト引数**: `q`, `limit`, `offset`など、関数の引数がそのままリクエストパラメータとして表示される

## 3. `Try it out`で動作確認

![](imgs\FastAPIのSwaggerUI画面説明3.png)

- **Try it outボタン**: これを押すとパラメータ入力欄が編集可能になる
- **Executeボタン**: APIリクエストを送信し、実際にレスポンスを確認できる

## 4. 実行後のリクエストとレスポンス

![](imgs\FastAPIのSwaggerUI画面説明4.png)

- **Curl**: Swagger UIが裏で実行したコマンド例。コピーしてターミナルでも試せる
- **Request URL**: 実際に送信されたリクエストの完全なURL
- **Server response**: Code（HTTPステータス）、Response body（JSONなどの本体）、Response headers（付帯情報）が確認できる

## 5. レスポンス仕様の詳細

![](imgs\FastAPIのSwaggerUI画面説明5.png)

- Code（ステータスコード）
    - 200: 成功時
    - 422: バリデーションエラー時
- Media type: application/json など、レスポンスの形式を示す
- Example Value: 実際に返ってくるデータのサンプル。200ならユーザー情報の配列、422ならエラーメッセージが表示される
- Links: OpenAPI仕様の機能の一つで、レスポンスの値を使って次のAPI呼び出しを関連付けられる

## まとめ

Swagger UIでは以下のことがわかる：

- API全体のメタ情報（タイトル・バージョン・説明）
- エンドポイントの一覧とリクエスト仕様
- Try it outでAPIの動作確認
- 実際のリクエストとレスポンスの詳細
- レスポンスのスキーマやサンプル

## 感想

FastAPIを学ぶ上で、Swagger UIを読み解くことは非常に有効だと感じた。コードとUIを照らし合わせながら理解すると、API設計の全体像が見えやすくなる