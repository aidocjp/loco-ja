+++
title = "認証"
description = ""
date = 2021-05-01T18:20:00+00:00
updated = 2021-05-01T18:20:00+00:00
draft = false
weight = 1
sort_by = "weight"
template = "docs/page.html"

[extra]
lead = ""
toc = true
top = false
flair =[]
+++

## ユーザーパスワード認証

`Loco`はユーザー認証プロセスを簡素化し、新しいWebサイトを素早くセットアップできるようにします。この機能は時間を節約するだけでなく、アプリケーションのコアロジックの作成に集中する柔軟性も提供します。

### 認証の設定

`auth`機能はライブラリのデフォルトとして付属しています。必要に応じて、これをオフにして認証を手動で処理することもできます。

### SaaSアプリで始める

[loco cli](/docs/getting-started/tour)を使用してアプリを作成し、`SaaS app (with DB and user auth)`オプションを選択します。

組み込みの認証コントローラーを確認するには、以下のコマンドを実行します：

```sh
$ cargo loco routes
 .
 .
 .
[POST] /api/auth/forgot
[POST] /api/auth/login
[POST] /api/auth/register
[POST] /api/auth/reset
[GET] /api/auth/verify
[GET] /api/auth/current
 .
 .
 .
```

### 新規ユーザーの登録

`/api/auth/register`エンドポイントは、アカウント検証用の`email_verification_token`と共に新しいユーザーをデータベースに作成します。検証リンクを含むウェルカムメールがユーザーに送信されます。

##### Example Curl Request:

```sh
curl --location '127.0.0.1:5150/api/auth/register' \
     --header 'Content-Type: application/json' \
     --data-raw '{
         "name": "Loco user",
         "email": "user@loco.rs",
         "password": "12341234"
     }'
```

セキュリティ上の理由から、ユーザーが既に登録されている場合、新しいユーザーは作成されず、ユーザーのメール詳細を公開することなく200ステータスが返されます。

### ログイン

新しいユーザーを登録した後、以下のリクエストを使用してログインします：

##### Example Curl Request:

```sh
curl --location '127.0.0.1:5150/api/auth/login' \
     --header 'Content-Type: application/json' \
     --data-raw '{
         "email": "user@loco.rs",
         "password": "12341234"
     }'
```

レスポンスには、認証用のJWTトークン、ユーザーID、名前、検証ステータスが含まれます。

```sh
{
    "token": "...",
    "pid": "2b20f998-b11e-4aeb-96d7-beca7671abda",
    "name": "Loco user",
    "is_verified": false
}
```

- **Token**: 認証エンドポイントへのリクエストを可能にするJWTトークン。デフォルトのトークン有効期限をカスタマイズし、環境間でシークレットが異なることを確認するには、[設定ドキュメント](@/docs/the-app/your-project.md)を参照してください。
- **pid** - 新しいユーザーを作成する際に生成される一意の識別子。
- **Name** - アカウントに関連付けられたユーザーの名前。
- **Is Verified** - ユーザーがアカウントを検証したかどうかを示すフラグ。

### アカウントの検証

ユーザー登録時に、検証リンク付きのメールが送信されます。このリンクを訪問すると、データベースの`email_verified_at`フィールドが更新され、ログインレスポンスの`is_verified`フラグがtrueに変更されます。

#### Example Curl request:

```sh
curl --location --request GET '127.0.0.1:5150/api/auth/verify/TOKEN' \
     --header 'Content-Type: application/json'
```

### パスワードリセットフロー

#### パスワードを忘れた場合

`forgot`エンドポイントは、ペイロードにユーザーのメールのみを必要とします。パスワードリセットリンク付きのメールが送信され、データベースに`reset_token`が設定されます。

##### Example Curl request:

```sh
curl --location '127.0.0.1:5150/api/auth/forgot' \
     --header 'Content-Type: application/json' \
     --data-raw '{
         "email": "user@loco.rs"
     }'
```

#### パスワードのリセット

パスワードをリセットするには、`forgot`エンドポイントで生成されたトークンと新しいパスワードを送信します。

##### Example Curl request:

```sh
curl --location '127.0.0.1:5150/api/auth/reset' \
     --header 'Content-Type: application/json' \
     --data '{
         "token": "TOKEN",
         "password": "new-password"
     }'
```

### 現在のユーザーを取得

このエンドポイントは認証ミドルウェアによって保護されています。

```sh
curl --location --request GET '127.0.0.1:5150/api/auth/current' \
     --header 'Content-Type: application/json' \
     --header 'Authorization: Bearer TOKEN'
```

### 認証されたエンドポイントの作成

認証されたエンドポイントを設定するには、`loco_rs`ライブラリから`controller::middleware`をインポートし、関数エンドポイントパラメータに認証ミドルウェアを組み込みます。

Rustでの以下の例を検討してください：

```rust
use axum::{extract::State, Json};
use loco_rs::{
    app::AppContext,
    controller::middleware,
    Result,
};

async fn current(
    auth: middleware::auth::JWT,
    State(ctx): State<AppContext>,
) -> Result<Response> {
    let user = users::Model::find_by_pid(&ctx.db, &auth.claims.pid).await?;
    /// Some response
}

```

## API認証

### 新しいアプリの作成

今回は、[loco cli](/docs/getting-started/tour)を使用してRESTアプリを作成し、`Rest app`オプションを選択します。
新しいアプリを作成するには、以下のコマンドを実行し、指示に従ってください：

```sh
$ loco new
```

組み込みの認証コントローラーを確認するには、以下のコマンドを実行します：

```sh
$ cargo loco routes
 .
 .
 .
[POST] /api/auth/forgot
[POST] /api/auth/login
[POST] /api/auth/register
[POST] /api/auth/reset
[GET] /api/auth/verify
[GET] /api/auth/current
 .
 .
 .
```

### 新規ユーザーの登録

`/api/auth/register`エンドポイントは、リクエスト認証用の`api_key`と共に新しいユーザーをデータベースに作成します。`api_key`は今後のリクエストでの認証に使用されます。

#### Example Curl Request:

```sh
curl --location '127.0.0.1:5150/api/auth/register' \
     --header 'Content-Type: application/json' \
     --data-raw '{
         "name": "Loco user",
         "email": "user@loco.rs",
         "password": "12341234"
     }'
```

新しいユーザーを登録した後、データベースで新しいユーザーの`api_key`が表示されていることを確認してください。

### API認証を使用した認証エンドポイントの作成

API認証エンドポイントを設定するには、loco_rsライブラリから`controller::middleware`をインポートし、`middleware::auth::ApiToken`を使用して関数エンドポイントパラメータに認証ミドルウェアを含めます。

Rustでの以下の例を検討してください：

```rust
use loco_rs::prelude::*;
use loco_rs::controller::middleware;
use crate::{models::_entities::users, views::user::CurrentResponse};

async fn current_by_api_key(
    auth: middleware::auth::ApiToken<users::Model>,
    State(_ctx): State<AppContext>,
) -> Result<Response> {
    format::json(CurrentResponse::new(&auth.user))
}

pub fn routes() -> Routes {
    Routes::new()
        .prefix("user")
        .add("/current-api", get(current_by_api_key))
}
```

### API認証エンドポイントへのリクエスト

認証されたエンドポイントにリクエストするには、`Authorization`ヘッダーに`API_KEY`を渡す必要があります。

#### Example Curl Request:

```sh
curl --location '127.0.0.1:5150/api/user/current-api' \
     --header 'Content-Type: application/json' \
     --header 'Authorization: Bearer API_KEY'
```

`API_KEY`が有効な場合、ユーザーの詳細を含むレスポンスが得られます。
