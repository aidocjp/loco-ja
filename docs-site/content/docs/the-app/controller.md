+++
title = "Controllers"
description = ""
date = 2021-05-01T18:10:00+00:00
updated = 2021-05-01T18:10:00+00:00
draft = false
weight = 5
sort_by = "weight"
template = "docs/page.html"

[extra]
lead = ""
toc = true
top = false
flair =[]
+++

`Loco`は[axum](https://crates.io/crates/axum)をラップしたフレームワークで、ルート、ミドルウェア、認証などを簡単に管理できる直接的なアプローチを開発箱で提供します。いつでも強力なaxum Routerを活用し、カスタムミドルウェアやルートで拡張することができます。

# コントローラーとルーティング


## コントローラーの追加

プロジェクトに接続されたスターターコントローラーの作成を簡素化する便利なコードジェネレーターを提供します。さらに、テストファイルも生成され、コントローラーのテストを簡単に行うことができます。

コントローラーを生成：

```sh
$ cargo loco generate controller [OPTIONS] <CONTROLLER_NAME>
```

コントローラーを生成した後、`src/controllers`の作成されたファイルに移動してコントローラーエンドポイントを確認してください。このコントローラーのテストについては、テスト（tests/requestsフォルダ内）のドキュメントも確認できます。


### アクティブなルートの表示

登録されているすべてのコントローラーのリストを表示するには、次のコマンドを実行してください：

```sh
$ cargo loco routes

[GET] /_health
[GET] /_ping
[POST] /auth/forgot
[POST] /auth/login
[POST] /auth/register
[POST] /auth/reset
[POST] /auth/verify
[GET] /auth/current
```

このコマンドは、システムに現在登録されているコントローラーの包括的な概要を提供します。

## AppRoutes

`AppRoutes`は`Loco`フレームワークの中核コンポーネントで、アプリケーションのルートを管理・整理するのに役立ちます。異なるコントローラーからルートを追加、プレフィックス付与、収集する便利な方法を提供します。

### 機能

- **ルートの追加**：異なるコントローラーからルートを簡単に追加できます。
- **ルートのプレフィックス**：ルートのグループに共通のプレフィックスを適用できます。
- **ルートの収集**：すべてのルートを単一のコレクションに集めて、さらなる処理を行えます。

### 例

#### ルートの追加

異なるコントローラーから`AppRoutes`にルートを追加できます：

```rust
use loco_rs::controller::AppRoutes;
use loco_rs::prelude::*;
use axum::routing::get;

fn routes(_ctx: &AppContext) -> AppRoutes {
  AppRoutes::empty()
          .add_route(Routes::new().add("/", get(home_handler)))
          .add_route(Routes::new().add("/about", get(about_handler)))
}
```

### ルートへのプレフィックス付与

ルートのグループに共通のプレフィックスを適用する：

```rust
use loco_rs::controller::AppRoutes;
use loco_rs::prelude::*;
use axum::routing::get;

fn routes(_ctx: &AppContext) -> AppRoutes {
    AppRoutes::empty()
        .prefix("/api")
        .add_route(Routes::new().add("/users", get(users_handler)))
        .add_route(Routes::new().add("/posts", get(posts_handler)))
}
```

### ルートのネスト

AppRoutesはルートのネストを可能にし、複雑なルート階層の整理と管理を簡単にします。
共通のプレフィックスを共有する関連ルートのセットがある場合に特に便利です。

```rust
 use loco_rs::controller::AppRoutes;
use loco_rs::prelude::*;
use axum::routing::get;

fn routes(_ctx: &AppContext) -> AppRoutes {
  let route = Routes::new().add("/", get(|| async { "notes" }));
  AppRoutes::with_default_routes()
        .prefix("api")
        .add_route(controllers::auth::routes())
        .nest_prefix("v1")
        .nest_route("/notes", route)
}
```

## 状態の追加

アプリのコンテキストと状態は`AppContext`に保持され、これはLocoが提供し設定するものです。アプリが起動するときにカスタムデータ、
ロジック、またはエンティティをロードして、すべてのコントローラーで利用できるようにしたい場合があります。

これはAxumの`Extension`を使用して実現できます。以下は時間のかかるタスクであるLLMモデルをロードし、それをコントローラーエンドポイントに提供する例で、モデルは既にロードされて使用可能な状態になっています。

まず、`src/app.rs`にライフサイクルフックを追加します：

```rust
    // in src/app.rs, in your Hooks trait impl override the `after_routes` hook:

    async fn after_routes(router: axum::Router, _ctx: &AppContext) -> Result<axum::Router> {
        // cache should reside at: ~/.cache/huggingface/hub
        println!("loading model");
        let model = Llama::builder()
            .with_source(LlamaSource::llama_7b_code())
            .build()
            .unwrap();
        println!("model ready");
        let st = Arc::new(RwLock::new(model));

        Ok(router.layer(Extension(st)))
    }
```

次に、好きな場所でこの状態拡張を使用します。以下はコントローラーエンドポイントの例です：

```rust
async fn candle_llm(Extension(m): Extension<Arc<RwLock<Llama>>>) -> impl IntoResponse {
    // use `m` from your state extension
    let prompt = "write binary search";
    ...
}
```

## グローバルなアプリ全体の状態 {#global-app-wide-state}

コントローラー、ワーカー、アプリの他の領域間で共有できる状態が必要な場合があります。

[shared-global-state](https://github.com/loco-rs/shared-global-state)アプリの例を参照して、C言語ベースの画像操作ライブラリである`libvips`を統合する方法を確認できます。`libvips`は開発者に奇妙な要求をします：アプリプロセス毎に単一のインスタンスをロードしたままにすることです。これは[単一の`lazy_static`フィールド](https://github.com/loco-rs/shared-global-state/blob/main/src/app.rs#L27-L34)を保持し、アプリの異なる場所から参照することで実現しています。

アプリの個々の部分でそれがどのように行われるかを以下を読んで確認してください。

### コントローラーでの共有状態

この文書で提供されているソリューションを使用できます。実際の例は[こちら](https://github.com/loco-rs/loco/blob/master/examples/llm-candle-inference/src/app.rs#L41)にあります。

### ワーカーでの共有状態

ワーカーは[アプリフック](https://github.com/loco-rs/loco/blob/master/starters/saas/src/app.rs#L59)で意図的にそのまま初期化されます。

これは、状態をフィールドとして受け取る「通常の」Rustの構造体として形作ることができることを意味します。そして、performでそのフィールドを参照します。

`shared-global-state`例でグローバル`vips`インスタンスを使用して[ワーカーがどのように初期化される](https://github.com/loco-rs/shared-global-state/blob/main/src/workers/downloader.rs#L19)かを示しています。

設計上、_コントローラーとワーカー間での状態共有は意味を持たない_ことに注意してください。なぜなら、最初は同じプロセスでワーカーとコントローラーを実行することを選択する（そして状態を共有する）かもしれませんが、水平スケールする際にはキューに支えられた適切なワーカーに素早く切り替え、独立したワーカープロセスで実行したくなるからです。したがって、あなた自身のためにも、ワーカーは設計上コントローラーと共有状態を持つべきではありません。

### タスクでの共有状態

タスクは、実行されるバイナリと同様のライフサイクルを持つため、共有状態に対して実際の価値を持ちません。プロセスが起動し、ブートし、必要なすべてのリソースを作成し（データベースへの接続など）、タスクロジックを実行してから終了します。 

## コントローラー内のルート

コントローラーはLocoのルート機能を定義します。以下の例では、コントローラーが1つのGETエンドポイントと1つのPOSTエンドポイントを作成しています：

```rust
use axum::routing::{get, post};
Routes::new()
    .add("/", get(hello))
    .add("/echo", post(echo))
```

`prefix`関数を使用してコントローラー内のすべてのルートに`prefix`を定義することもできます。

## レスポンスの送信

レスポンス送信者は`format`モジュールにあります。ルートからレスポンスを送信するいくつかの方法を以下に示します：

```rust

// keep a best practice of returning a `Result<impl IntoResponse>` to be able to swap return types transparently
pub async fn list(...) -> Result<impl IntoResponse> // ..

// use `json`, `html` or `text` for simple responses
format::json(item)


// use `render` for a builder interface for more involved responses. you can still terminate with
// `json`, `html`, or `text`
format::render()
    .etag("foobar")?
    .json(Entity::find().all(&ctx.db).await?)
```

### コンテンツタイプ対応レスポンス

フォーマットタイプが検出されて渡される
レスポンダーメカニズムにオプトインできます。

これには`Format`エクストラクターを使用します：

```rust
pub async fn get_one(
    respond_to: RespondTo,
    Path(id): Path<i32>,
    State(ctx): State<AppContext>,
) -> Result<Response> {
    let res = load_item(&ctx, id).await?;
    match respond_to {
        RespondTo::Html => format::html(&format!("<html><body>{:?}</body></html>", item.title)),
        _ => format::json(item),
    }
}
```

### カスタムエラー

異なるフォーマットに基づいて異なるレンダリングを行い、さらに
受け取ったエラーの種類に基づいて異なるレンダリングを行いたい場合の例です。


```rust
pub async fn get_one(
    respond_to: RespondTo,
    Path(id): Path<i32>,
    State(ctx): State<AppContext>,
) -> Result<Response> {
    // having `load_item` is useful because inside the function you can call and use
    // '?' to bubble up errors, then, in here, we centralize handling of errors.
    // if you want to freely use code statements with no wrapping function, you can
    // use the experimental `try` feature in Rust where you can do:
    // ```
    // let res = try {
    //     ...
    //     ...
    // }
    //
    // match res { ..}
    // ```
    let res = load_item(&ctx, id).await;

    match res {
        // we're good, let's render the item based on content type
        Ok(item) => match respond_to {
            RespondTo::Html => format::html(&format!("<html><body>{:?}</body></html>", item.title)),
            _ => format::json(item),
        },
        // we have an opinion how to render out validation errors, only in HTML content
        Err(Error::Model(ModelError::Validation(errors))) => match respond_to {
            RespondTo::Html => {
                format::html(&format!("<html><body>errors: {errors:?}</body></html>"))
            }
            _ => bad_request("opaque message: cannot respond!"),
        },
        // we have no clue what this is, let the framework render default errors
        Err(err) => Err(err),
    }
}
```

ここでは、ワークフローをまず関数でラップし、結果タイプを取得することでエラーハンドリングも「集中化」しています。

次に、2レベルのマッチを作成します：

1. 結果タイプをマッチ
2. フォーマットタイプをマッチ

処理の知識が不足している場所では、エラーをそのまま返し、フレームワークにデフォルトエラーをレンダリングさせます。

## コントローラーの手動作成

#### 1. コントローラーファイルの作成

`src/controllers`パスの下に新しいファイルを作成することから始めます。例えば、`example.rs`という名前のファイルを作成しましょう。

#### 2. mod.rsでファイルをロード

`src/controllers`フォルダー内の`mod.rs`ファイルで新しく作成したコントローラーファイルをロードすることを確認してください。

#### 3. App Hooksでコントローラーを登録

Appフックの実装（例：App構造体）で、コントローラーの`Routes`を`AppRoutes`に追加します：

```rust
// src/app.rs

pub struct App;
#[async_trait]
impl Hooks for App {
    fn routes() -> AppRoutes {
        AppRoutes::with_default_routes().prefix("prefix")
            .add_route(controllers::example::routes())
    }
    ...
}

```

# ミドルウェア

Locoは箱から出してすぐに使えるビルトインミドルウェアのセットが付属しています。一部はデフォルトで有効になっていますが、他は設定が必要です。ミドルウェアの登録は柔軟で、`*.yaml`環境設定またはコード内で直接管理できます。

## デフォルトスタック

有効なすべてのミドルウェアを取得するには以下のコマンドを実行します
<!-- <snip id="cli-middleware-list" inject_from="yaml" template="sh"> -->
```sh
cargo loco middleware --config
```
<!-- </snip> -->

これは`development`モードでのスタックです：

```sh
$ cargo loco middleware --config

limit_payload          {"body_limit":{"Limit":1000000}}
cors                   {"enable":true,"allow_origins":["any"],"allow_headers":["*"],"allow_methods":["*"],"max_age":null,"vary":["origin","access-control-request-method","access-control-request-headers"]}
catch_panic            {"enable":true}
etag                   {"enable":true}
logger                 {"config":{"enable":true},"environment":"development"}
request_id             {"enable":true}
fallback               {"enable":true,"code":200,"file":null,"not_found":null}
powered_by             {"ident":"loco.rs"}


remote_ip              (disabled)
compression            (disabled)
timeout                (disabled)
static_assets          (disabled)
secure_headers         (disabled)
```

### 例：すべてのミドルウェアを無効化

有効になっているものを取り、関連するフィールドで`enable: false`を使用します。`server`内の`middlewares:`セクションがない場合は、追加してください。

```yaml
server:
  middlewares:
    cors:
      enable: false
    catch_panic:
      enable: false
    etag:
      enable: false
    logger:
      enable: false
    request_id:
      enable: false
    fallback:
      enable: false
```

結果：

```sh
$ cargo loco middleware --config
powered_by             {"ident":"loco.rs"}


cors                   (disabled)
catch_panic            (disabled)
etag                   (disabled)
remote_ip              (disabled)
compression            (disabled)
timeout_request        (disabled)
static                 (disabled)
secure_headers         (disabled)
logger                 (disabled)
request_id             (disabled)
fallback               (disabled)
```

`server.ident`の値を変更することで`powered_by`ミドルウェアを制御できます：

```yaml
server:
    ident: my-server #(or empty string to disable)
```

### 例：デフォルト以外のミドルウェアを追加

_Remote IP_ミドルウェアをスタックに追加しましょう。これは設定だけで行えます：

```yaml
server:
  middlewares:
    remote_ip:
      enable: true
```

結果：

```sh
$ cargo loco middleware --config

limit_payload          {"body_limit":{"Limit":1000000}}
cors                   {"enable":true,"allow_origins":["any"],"allow_headers":["*"],"allow_methods":["*"],"max_age":null,"vary":["origin","access-control-request-method","access-control-request-headers"]}
catch_panic            {"enable":true}
etag                   {"enable":true}
remote_ip              {"enable":true,"trusted_proxies":null}
logger                 {"config":{"enable":true},"environment":"development"}
request_id             {"enable":true}
fallback               {"enable":true,"code":200,"file":null,"not_found":null}
powered_by             {"ident":"loco.rs"}
```

### 例：有効なミドルウェアの設定を変更

リクエストボディの制限を`5mb`に変更しましょう。ミドルウェアの設定をオーバーライドするときは、`enable: true`を保持することを忘れないでください：

```yaml
  middlewares:
    limit_payload:
      body_limit: 5mb
```

結果：

```sh
$ cargo loco middleware --config

limit_payload          {"body_limit":{"Limit":5000000}}
cors                   {"enable":true,"allow_origins":["any"],"allow_headers":["*"],"allow_methods":["*"],"max_age":null,"vary":["origin","access-control-request-method","access-control-request-headers"]}
catch_panic            {"enable":true}
etag                   {"enable":true}
logger                 {"config":{"enable":true},"environment":"development"}
request_id             {"enable":true}
fallback               {"enable":true,"code":200,"file":null,"not_found":null}
powered_by             {"ident":"loco.rs"}


remote_ip              (disabled)
compression            (disabled)
timeout_request        (disabled)
static                 (disabled)
secure_headers         (disabled)
```

### 認証
`Loco`フレームワークでは、ミドルウェアが認証において重要な役割を果たします。`Loco`は、JSON Web Token（JWT）とAPIキー認証を含む様々な認証方法をサポートしています。このセクションでは、アプリケーションで認証ミドルウェアを設定して使用する方法を説明します。

#### JSON Web Token (JWT)

##### 設定
デフォルトでは、LocoはJWTにBearer認証を使用します。ただし、設定ファイルのauth.jwtセクションでこの動作をカスタマイズできます。
* *Bearer認証：* 設定を空白のままにするか、次のように明示的に設定します：
  ```yaml
  # Authentication Configuration
  auth:
    # JWT authentication
    jwt:
      location: Bearer
  ...
  ```
* *Cookie認証：* トークンを抽出する場所を設定し、Cookieの名前を指定します：
  ```yaml
  # Authentication Configuration
  auth:
    # JWT authentication
    jwt:
      location: 
        from: Cookie
        name: token
  ...
  ```
* *クエリパラメータ認証：* クエリパラメータの場所と名前を指定します：
  ```yaml
  # Authentication Configuration
  auth:
    # JWT authentication
    jwt:
      location: 
        from: Query
        name: token
  ...
  ```

##### 使用方法
コントローラーのパラメータで、認証に`auth::JWT`を使用します。これにより、設定に基づいた認証検証がトリガーされます。
```rust
use loco_rs::prelude::*;

async fn current(
    auth: auth::JWT,
    State(_ctx): State<AppContext>,
) -> Result<Response> {
    // Your implementation here
}
```
さらに、auth::JWTを`auth::ApiToken<users::Model>`に置き換えることで、現在のユーザーを取得できます。

#### APIキー
APIキー認証の場合は、auth::ApiTokenを使用します。このミドルウェアはAPIキーをユーザーデータベースレコードと照合し、対応するユーザーを認証パラメータにロードします。
```rust
use loco_rs::prelude::*;

async fn current(
    auth: auth::ApiToken<users::Model>,
    State(_ctx): State<AppContext>,
) -> Result<Response> {
    // Your implementation here
}
```

## Catch Panic

このミドルウェアは、アプリケーションでのリクエスト処理中に発生するパニックをキャッチします。パニックが発生すると、エラーをログに記録し、内部サーバーエラーレスポンスを返します。このミドルウェアは、サーバーをクラッシュさせることなく、アプリケーションが予期しないエラーを適切に処理できるようにします。

ミドルウェアを無効にするには、次のように設定を編集します：

```yaml
#...
  middlewares:
    catch_panic:
      enable: false
```


## Limit Payload

Limit Payloadミドルウェアは、HTTPリクエストペイロードの最大許容サイズを制限します。デフォルトでは有効になっており、2MBの制限で設定されています。

設定ファイルでこの動作をカスタマイズまたは無効にできます。

### カスタム制限の設定
```yaml
#...
  middlewares:
    limit_payload:
      body_limit: 5mb
```

### ペイロードサイズ制限の無効化
制限を完全に削除するには、`body_limit`を`disable`に設定します：
```yaml
#...
  middlewares:
    limit_payload:
      body_limit: disable
```


##### 使用方法
コントローラーのパラメータで`axum::body::Bytes`を使用します。
```rust
use loco_rs::prelude::*;

async fn current(_body: axum::body::Bytes,) -> Result<Response> {
    // Your implementation here
}
```

## Timeout

アプリケーションで処理されるリクエストにタイムアウトを適用します。このミドルウェアは、リクエストが指定されたタイムアウト期間を超えて実行されないようにし、アプリケーションの全体的なパフォーマンスと応答性を向上させます。

リクエストが指定されたタイムアウト期間を超えた場合、ミドルウェアはクライアントに`408 Request Timeout`ステータスコードを返し、リクエストの処理に時間がかかりすぎたことを示します。

ミドルウェアを有効にするには、次のように設定を編集します：

```yaml
#...
  middlewares:
    timeout_request:
      enable: false
      timeout: 5000
```


## Logger

HTTPリクエストのロギング機能を提供します。HTTPメソッド、URI、バージョン、ユーザーエージェント、関連するリクエストIDなど、各リクエストに関する詳細情報を記録します。さらに、アプリケーションのランタイム環境をログコンテキストに統合し、環境固有のロギング（例：「development」、「production」）を可能にします。

ミドルウェアを無効にするには、次のように設定を編集します：

```yaml
#...
  middlewares:
    logger:
      enable: false
```


## Fallback

SaaSスターター（またはAPIファーストではないスターター）を選択すると、_Locoウェルカムスクリーン_を使用したデフォルトのフォールバック動作が得られます。これは開発専用モードで、`404`リクエストが発生すると、何が起こったか、次に何をすべきかを示す親切でフレンドリーなページが表示されます。これは静的ハンドラーよりも優先されるため、静的コンテンツを提供したい場合は無効にする必要があります。

`development.yaml`ファイルでこの動作を無効化またはカスタマイズできます。いくつかのオプションを設定できます：


```yaml
# the default pre-baked welcome screen
fallback:
    enable: true
```

```yaml
# a different predefined 404 page
fallback:
    enable: true
    file: assets/404.html
```

```yaml
# a message, and customizing the status code to return 200 instead of 404
fallback:
    enable: true
    code: 200
    not_found: cannot find this resource
```

本番環境では、これを無効にすることを推奨します。

```yaml
# disable. you can also remove the `fallback` section entirely to disable
fallback:
    enable: false
```

## Remote IP

When your app is under a proxy or a load balancer (e.g. Nginx, ELB, etc.), it does not face the internet directly, which is why if you want to find out the connecting client IP, you'll get a socket which indicates an IP that is actually your load balancer instead.

The load balancer or proxy is responsible for doing the socket work against the real client IP, and then giving your app the load via the proxy back connection to your app.

This is why when your app has a concrete business need for getting the real client IP you need to use the de-facto standard proxies and load balancers use for handing you this information: the `X-Forwarded-For` header.

Loco provides the `remote_ip` section for configuring the `RemoteIP` middleware:

```yaml
server:
  middleware:
    # calculate remote IP based on `X-Forwarded-For` when behind a proxy or load balancer
    # use RemoteIP(..) extractor to get the remote IP.
    # without this middleware, you'll get the proxy IP instead.
    # For more: https://github.com/rails/rails/blob/main/actionpack/lib/action_dispatch/middleware/remote_ip.rb
    #
    # NOTE! only enable when under a proxy, otherwise this can lead to IP spoofing vulnerabilities
    # trust me, you'll know if you need this middleware.
    remote_ip:
      enable: true
      # # replace the default trusted proxies:
      # trusted_proxies:
      # - ip range 1
      # - ip range 2 ..
    # Generating a unique request ID and enhancing logging with additional information such as the start and completion of request processing, latency, status code, and other request details.
```

Then, use the `RemoteIP` extractor to get the IP:

```rust
#[debug_handler]
pub async fn list(ip: RemoteIP, State(ctx): State<AppContext>) -> Result<Response> {
    println!("remote ip {ip}");
    format::json(Entity::find().all(&ctx.db).await?)
}
```

When using the `RemoteIP` middleware, take note of the security implications vs. your current architecture (as noted in the documentation and in the configuration section): if your app is NOT under a proxy, you can be prone to IP spoofing vulnerability because anyone can set headers to arbitrary values, and specifically, anyone can set the `X-Forwarded-For` header.

This middleware is not enabled by default. Usually, you *will know* if you need this middleware and you will be aware of the security aspects of using it in the correct architecture. If you're not sure -- don't use it (keep `enable` to `false`).


## Secure Headers

Loco comes with default secure headers applied by the `secure_headers` middleware. This is similar to what is done in the Rails ecosystem with [secure_headers](https://github.com/github/secure_headers).

In your `server.middleware` YAML section you will find the `github` preset by default (which is what Github and Twitter recommend for secure headers).

```yaml
server:
  middleware:
    # set secure headers
    secure_headers:
      preset: github
```

You can also override select headers:

```yaml
server:
  middleware:
    # set secure headers
    secure_headers:
      preset: github
      overrides:
        foo: bar
```

Or start from scratch:

```yaml
server:
  middleware:
    # set secure headers
    secure_headers:
      preset: empty
      overrides:
        foo: bar
```

To support `htmx`, You can add the following override, to allow some inline running of scripts:

```yaml
secure_headers:
    preset: github
    overrides:
        # this allows you to use HTMX, and has unsafe-inline. Remove or consider in production
        "Content-Security-Policy": "default-src 'self' https:; font-src 'self' https: data:; img-src 'self' https: data:; object-src 'none'; script-src 'unsafe-inline' 'self' https:; style-src 'self' https: 'unsafe-inline'"
```

## Compression

`Loco` leverages [CompressionLayer](https://docs.rs/tower-http/0.5.0/tower_http/compression/index.html) to enable a `one click` solution.

To enable response compression, based on `accept-encoding` request header, simply edit the configuration as follows:

```yaml
#...
  middlewares:
    compression:
      enable: true
```

Doing so will compress each response and set `content-encoding` response header accordingly.

## Precompressed assets


`Loco` leverages [ServeDir::precompressed_gzip](https://docs.rs/tower-http/latest/tower_http/services/struct.ServeDir.html#method.precompressed_gzip) to enable a `one click` solution of serving pre compressed assets.

If a static assets exists on the disk as a `.gz` file, `Loco` will serve it instead of compressing it on the fly.

```yaml
#...
middlewares:
  ...
  static_assets:
    ...
    precompressed: true
```

## CORS
This middleware enables Cross-Origin Resource Sharing (CORS) by allowing configurable origins, methods, and headers in HTTP requests. 
It can be tailored to fit various application requirements, supporting permissive CORS or specific rules as defined in the middleware configuration.

```yaml
#...
middlewares:
  ...
  cors:
    enable: true
    # Set the value of the [`Access-Control-Allow-Origin`][mdn] header
    # allow_origins:
    #   - https://loco.rs
    # Set the value of the [`Access-Control-Allow-Headers`][mdn] header
    # allow_headers:
    # - Content-Type
    # Set the value of the [`Access-Control-Allow-Methods`][mdn] header
    # allow_methods:
    #   - POST
    # Set the value of the [`Access-Control-Max-Age`][mdn] header in seconds
    # max_age: 3600

```

## Handler and Route based middleware

`Loco` also allow us to apply [layers](https://docs.rs/tower/latest/tower/trait.Layer.html) to specific handlers or
routes.
For more information on handler and route based middleware, refer to the [middleware](/docs/the-app/controller/#middleware)
documentation.


### Handler based middleware:

Apply a layer to a specific handler using `layer` method.

```rust
// src/controllers/auth.rs
pub fn routes() -> Routes {
    Routes::new()
        .prefix("auth")
        .add("/register", post(register).layer(middlewares::log::LogLayer::new()))
}
```

### Route based middleware:

Apply a layer to a specific route using `layer` method.

```rust
// src/main.rs
pub struct App;

#[async_trait]
impl Hooks for App {
    fn routes(_ctx: &AppContext) -> AppRoutes {
        AppRoutes::with_default_routes()
            .add_route(
                controllers::auth::routes()
                    .layer(middlewares::log::LogLayer::new()),
            )
    }
}
```

# Request Validation
`JsonValidate` extractor simplifies input [validation](https://github.com/Keats/validator) by integrating with the validator crate. Here's an example of how to validate incoming request data:

### Define Your Validation Rules
```rust
use axum::debug_handler;
use loco_rs::prelude::*;
use serde::Deserialize;
use validator::Validate;

#[derive(Debug, Deserialize, Validate)]
pub struct DataParams {
    #[validate(length(min = 5, message = "custom message"))]
    pub name: String,
    #[validate(email)]
    pub email: String,
}
```
### Create a Handler with Validation
```rust
use axum::debug_handler;
use loco_rs::prelude::*;

#[debug_handler]
pub async fn index(
    State(_ctx): State<AppContext>,
    JsonValidate(params): JsonValidate<DataParams>,
) -> Result<Response> {
    format::empty()
}
```
Using the `JsonValidate` extractor, Loco automatically performs validation on the DataParams struct:
* If validation passes, the handler continues execution with params.
* If validation fails, a 400 Bad Request response is returned.

### Returning Validation Errors as JSON
If you'd like to return validation errors in a structured JSON format, use `JsonValidateWithMessage` instead of `JsonValidate`. The response format will look like this:

```json
{
  "errors": {
    "email": [
      {
        "code": "email",
        "message": null,
        "params": {
          "value": "ad"
        }
      }
    ],
    "name": [
      {
        "code": "length",
        "message": "custom message",
        "params": {
          "min": 5,
          "value": "d"
        }
      }
    ]
  }
}
```  

# Pagination

In many scenarios, when querying data and returning responses to users, pagination is crucial. In `Loco`, we provide a straightforward method to paginate your data and maintain a consistent pagination response schema for your API responses.

We assume you have a `notes` entity and/or scaffold (replace this with any entity you like).

## Using pagination

```rust
use loco_rs::prelude::*;

let res = query::fetch_page(&ctx.db, notes::Entity::find(), &query::PaginationQuery::page(2)).await;
```


## Using pagination With Filter
```rust
use loco_rs::prelude::*;

let pagination_query = query::PaginationQuery {
    page_size: 100,
    page: 1,
};

let condition = query::condition().contains(notes::Column::Title, "loco");
let paginated_notes = query::paginate(
    &ctx.db,
    notes::Entity::find(),
    Some(condition.build()),
    &pagination_query,
)
.await?;
```

- Start by defining the entity you want to retrieve.
- Create your query condition (in this case, filtering rows that contain "loco" in the title column).
- Define the pagination parameters.
- Call the paginate function.

### Pagination view
After creating getting the `paginated_notes` in the previous example, you can choose which fields from the model you want to return and keep the same pagination response in all your different data responses.

Define the data you're returning to the user in Loco views. If you're not familiar with views, refer to the [documentation](@/docs/the-app/views.md) for more context.


Create a notes view file in `src/view/notes` with the following code:

```rust
use loco_rs::{
    controller::views::pagination::{Pager, PagerMeta},
    prelude::model::query::PaginatedResponse,
};
use serde::{Deserialize, Serialize};

use crate::models::_entities::notes;

#[derive(Debug, Deserialize, Serialize)]
pub struct ListResponse {
    id: i32,
    title: Option<String>,
    content: Option<String>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct PaginationResponse {}

impl From<notes::Model> for ListResponse {
    fn from(note: notes::Model) -> Self {
        Self {
            id: note.id.clone(),
            title: note.title.clone(),
            content: note.content,
        }
    }
}

impl PaginationResponse {
    #[must_use]
    pub fn response(data: PaginatedResponse<notes::Model>, pagination_query: &PaginationQuery) -> Pager<Vec<ListResponse>> {
        Pager {
            results: data
                .page
                .into_iter()
                .map(ListResponse::from)
                .collect::<Vec<ListResponse>>(),
            info: PagerMeta {
                page: pagination_query.page,
                page_size: pagination_query.page_size,
                total_pages: data.total_pages,
                total_items: data.total_items,
            },
        }
    }
}
```


# Testing
When testing controllers, the goal is to call the router's controller endpoint and verify the HTTP response, including the status code, response content, headers, and more.

To initialize a test request, use `use loco_rs::testing::prelude::*;`, which prepares your app routers, providing the request instance and the application context.

In the following example, we have a POST endpoint that returns the data sent in the POST request.

```rust
use loco_rs::testing::prelude::*;

#[tokio::test]
#[serial]
async fn can_print_echo() {
    configure_insta!();

    request::<App, _, _>(|request, _ctx| async move {
        let response = request
            .post("/example")
            .json(&serde_json::json!({"site": "Loco"}))
            .await;

        assert_debug_snapshot!((response.status_code(), response.text()));
    })
    .await;
}
```

As you can see initialize the testing request and using `request` instance calling /example endpoing.
the request returns a `Response` instance with the status code and the response test


## Async
When writing async tests with database data, it's important to ensure that one test does not affect the data used by other tests. Since async tests can run concurrently on the same database dataset, this can lead to unstable test results.

Instead of using `request`, as described in the documentation for synchronous tests, use the `request_with_create_db` function. This function generates a random database schema name and ensures that the tables are deleted once the test is completed.

Note: If you cancel the test run midway (e.g., by pressing `Ctrl + C`), the cleanup process will not execute, and the database tables will remain. In such cases, you will need to manually remove them.

```rust
use loco_rs::testing::prelude::*;

#[tokio::test]
async fn can_print_echo() {
    configure_insta!();

    request_with_create_db::<App, _, _>(|request, _ctx| async move {
        let response = request
            .post("/example")
            .json(&serde_json::json!({"site": "Loco"}))
            .await;

        assert_debug_snapshot!((response.status_code(), response.text()));
    })
    .await;
}
```

## Authenticated Endpoints
The following example works for both JWT and API_KEY Authentication.
```rust
use loco_rs::testing::prelude::*;
use super::prepare_data;

#[tokio::test]
#[serial]
async fn can_get_current_user() {
    configure_insta!();

    request::<App, _, _>(|request, ctx| async move {
        // Initialize the user
        let user = prepare_data::init_user_login(&request, &ctx).await;
        let (auth_key, auth_value) = prepare_data::auth_header(&user.token);

        // Then add the key to the request, usually in the header
        let response = request
            .get("/example")
            .add_header(auth_key, auth_value)
            .await;

        assert_eq!(
            response.status_code(),
            200,
            "Current request should succeed"
        );

        assert_debug_snapshot!((response.status_code(), response.text()));
    })
    .await;
}
```
