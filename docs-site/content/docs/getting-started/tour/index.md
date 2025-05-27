+++
title = "クイックツアー"
date = 2021-05-01T08:00:00+00:00
updated = 2021-05-01T08:00:00+00:00
draft = false
weight = 2
sort_by = "weight"
template = "docs/page.html"

[extra]
toc = true
top = false
flair =[]
+++


<img style="width:100%; max-width:640px" src="tour.png"/>
<br/>
<br/>
<br/>
Locoでブログバックエンドをわずか数分で作成してみましょう。まず、`loco`と`sea-orm-cli`をインストールします：

<!-- <snip id="quick-installation-command" inject_from="yaml" template="sh"> -->
```sh
cargo install loco
cargo install sea-orm-cli # Only when DB is needed
```
<!-- </snip> -->


新しいアプリを作成できます（"`SaaS` app"を選択）。クライアントサイドレンダリング付きのSaaSアプリを選択してください：

<!-- <snip id="loco-cli-new-from-template" inject_from="yaml" template="sh"> -->
```sh
❯ loco new
✔ ❯ App name? · myapp
✔ ❯ What would you like to build? · Saas App with client side rendering
✔ ❯ Select a DB Provider · Sqlite
✔ ❯ Select your background worker type · Async (in-process tokio async tasks)

🚂 Loco app generated successfully in:
myapp/

- assets: You've selected `clientside` for your asset serving configuration.

Next step, build your frontend:
  $ cd frontend/
  $ npm install && npm run build
```
<!-- </snip> -->

以下のような構成になります：

* データベースには`sqlite`を使用。データベースプロバイダについては、_models_セクションの[Sqlite vs Postgres](@/docs/the-app/models.md#sqlite-vs-postgres)で詳しく学べます。
* バックグラウンドワーカーには`async`を使用。ワーカー設定については、_workers_セクションで詳しく学べます。
* クライアントサイドアセット配信設定。つまり、バックエンドがAPIとして機能し、静的なクライアントサイドコンテンツも配信します。


`myapp`ディレクトリに移動し、`cargo loco start`を実行してアプリを起動します：
 
 
 <div class="infobox">
 クライアントサイドアセット配信オプションを設定している場合は、サーバーを起動する前にフロントエンドをビルドしてください。これは、frontendディレクトリに移動（`cd frontend`）して、`pnpm install`と`pnpm build`を実行することで行えます。
 </div>

<!-- <snip id="starting-the-server-command-with-output" inject_from="yaml" template="sh"> -->
```sh
$ cargo loco start

                      ▄     ▀
                                ▀  ▄
                  ▄       ▀     ▄  ▄ ▄▀
                                    ▄ ▀▄▄
                        ▄     ▀    ▀  ▀▄▀█▄
                                          ▀█▄
▄▄▄▄▄▄▄  ▄▄▄▄▄▄▄▄▄   ▄▄▄▄▄▄▄▄▄▄▄ ▄▄▄▄▄▄▄▄▄ ▀▀█
██████  █████   ███ █████   ███ █████   ███ ▀█
██████  █████   ███ █████   ▀▀▀ █████   ███ ▄█▄
██████  █████   ███ █████       █████   ███ ████▄
██████  █████   ███ █████   ▄▄▄ █████   ███ █████
██████  █████   ███  ████   ███ █████   ███ ████▀
  ▀▀▀██▄ ▀▀▀▀▀▀▀▀▀▀  ▀▀▀▀▀▀▀▀▀▀  ▀▀▀▀▀▀▀▀▀▀ ██▀
      ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀
                https://loco.rs

listening on port 5150
```
<!-- </snip> -->


<div class="infobox">
`cargo`を通して実行する必要はありませんが、開発時には強く推奨されます。`--release`でビルドする場合、バイナリにはコードを含むすべてが含まれており、`cargo`やRustは不要です。
</div>

## CRUD APIの追加

ユーザー認証機能を持つベースのSaaSアプリが生成されました。`scaffold`を使用して`post`と完全なCRUD APIを追加し、ブログバックエンドにしてみましょう：

<div class="infobox">
`--api`、`--html`、`--htmx`フラグを使用して、それぞれ`api`、`html`、`htmx`スキャフォールドの生成を選択できます。
</div>

クライアント側のコードベースを持つバックエンドを構築しているため、`--api`を使用してAPIを構築します：

```sh
$ cargo loco generate scaffold post title:string content:text --api

  :
  :
added: "src/controllers/post.rs"
injected: "src/controllers/mod.rs"
injected: "src/app.rs"
added: "tests/requests/post.rs"
injected: "tests/requests/mod.rs"
* Migration for `post` added! You can now apply it with `$ cargo loco db migrate`.
* A test for model `posts` was added. Run with `cargo test`.
* Controller `post` was added successfully.
* Tests for controller `post` was added successfully. Run `cargo test`.
```

データベースがマイグレーションされ、モデル、エンティティ、完全なCRUDコントローラーが自動的に生成されました。

アプリを再度起動します：
<!-- <snip id="starting-the-server-command-with-output" inject_from="yaml" template="sh"> -->
```sh
$ cargo loco start

                      ▄     ▀
                                ▀  ▄
                  ▄       ▀     ▄  ▄ ▄▀
                                    ▄ ▀▄▄
                        ▄     ▀    ▀  ▀▄▀█▄
                                          ▀█▄
▄▄▄▄▄▄▄  ▄▄▄▄▄▄▄▄▄   ▄▄▄▄▄▄▄▄▄▄▄ ▄▄▄▄▄▄▄▄▄ ▀▀█
██████  █████   ███ █████   ███ █████   ███ ▀█
██████  █████   ███ █████   ▀▀▀ █████   ███ ▄█▄
██████  █████   ███ █████       █████   ███ ████▄
██████  █████   ███ █████   ▄▄▄ █████   ███ █████
██████  █████   ███  ████   ███ █████   ███ ████▀
  ▀▀▀██▄ ▀▀▀▀▀▀▀▀▀▀  ▀▀▀▀▀▀▀▀▀▀  ▀▀▀▀▀▀▀▀▀▀ ██▀
      ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀
                https://loco.rs

listening on port 5150
```
<!-- </snip> -->

<div class="infobox"> 
選択したスキャフォールドテンプレートオプション（`--api`、`--html`、`--htmx`）によって、スキャフォールドリソースを作成する手順が変わります。`--api`フラグまたは`--htmx`フラグでは、以下の例を使用できます。しかし、`--html`フラグの場合は、ブラウザでpost作成手順を実行することをお勧めします。
  
`--html`スキャフォールドをテストするために`curl`を使用したい場合は、デフォルトでContent-Type `application/x-www-form-urlencoded`を使用し、ボディを`title=Your+Title&content=Your+Content`として送信する必要があります。必要に応じて、コード内でContent-Typeとして`application/json`を許可するよう変更できます。
</div>

次に、`curl`を使用して`post`を追加してみましょう：

```sh
$ curl -X POST -H "Content-Type: application/json" -d '{
  "title": "Your Title",
  "content": "Your Content xxx"
}' localhost:5150/api/posts
```

投稿一覧を取得できます：

```sh
$ curl localhost:5150/api/posts
```

振り返ると、ブログバックエンドを作成するコマンドは以下の通りでした：

1. `cargo install loco`
2. `cargo install sea-orm-cli`
3. `loco new`
4. `cargo loco generate scaffold post title:string content:text --api`

完了！`loco`での旅をお楽しみください 🚂

## SaaS認証機能の確認

生成されたアプリには、JWTベースの完全に動作する認証システムが含まれています。

### 新規ユーザー登録

`/api/auth/register`エンドポイントは、アカウント認証用の`email_verification_token`とともに新しいユーザーをデータベースに作成します。確認リンク付きのウェルカムメールがユーザーに送信されます。

```sh
$ curl --location 'localhost:5150/api/auth/register' \
     --header 'Content-Type: application/json' \
     --data-raw '{
         "name": "Loco user",
         "email": "user@loco.rs",
         "password": "12341234"
     }'
```

セキュリティ上の理由により、ユーザーが既に登録されている場合、新しいユーザーは作成されず、ユーザーのメールアドレスの詳細を公開することなく200ステータスが返されます。

### ログイン

新しいユーザーを登録した後、以下のリクエストを使用してログインします：

```sh
$ curl --location 'localhost:5150/api/auth/login' \
     --header 'Content-Type: application/json' \
     --data-raw '{
         "email": "user@loco.rs",
         "password": "12341234"
     }'
```

レスポンスには、認証用のJWTトークン、ユーザーID、名前、および認証ステータスが含まれます。

```sh
{
    "token": "...",
    "pid": "2b20f998-b11e-4aeb-96d7-beca7671abda",
    "name": "Loco user",
    "claims": null
    "is_verified": false
}
```

クライアントサイドアプリでは、このJWTトークンを保存し、認証されたリクエストを行うために_bearer token_（下記参照）を使用して以降のリクエストを行います。

### 現在のユーザー取得

このエンドポイントは認証ミドルウェアによって保護されています。先ほど取得したトークンを使用して、_bearer token_技術でリクエストを実行します（`TOKEN`を先ほど取得したJWTトークンに置き換えてください）：

```sh
$ curl --location --request GET 'localhost:5150/api/auth/current' \
     --header 'Content-Type: application/json' \
     --header 'Authorization: Bearer TOKEN'
```

これが最初の認証済みリクエストになります！

`controllers/auth.rs`のソースコードを確認して、自分のコントローラーで認証ミドルウェアを使用する方法を学んでください。
