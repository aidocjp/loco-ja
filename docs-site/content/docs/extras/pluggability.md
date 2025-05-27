+++
title = "プラグイン機能"
description = ""
date = 2021-05-01T18:10:00+00:00
updated = 2021-05-01T18:10:00+00:00
draft = false
weight = 3
sort_by = "weight"
template = "docs/page.html"

[extra]
lead = ""
toc = true
top = false
flair =[]
+++

## エラーレベルとオプション

おさらいとして、エラーレベルとそのロギングは`development.yaml`で制御できます：

### ロガー

<!-- <snip id="configuration-logger" inject_from="code" template="yaml"> -->

```yaml
# Application logging configuration
logger:
  # Enable or disable logging.
  enable: true
  # Enable pretty backtrace (sets RUST_BACKTRACE=1)
  pretty_backtrace: true
  # Log level, options: trace, debug, info, warn or error.
  level: debug
  # Define the logging format. options: compact, pretty or json
  format: compact
  # By default the logger has filtering only logs that came from your code or logs that came from `loco` framework. to see all third party libraries
  # Uncomment the line below to override to see all third party libraries you can enable this config and override the logger filters.
  # override_filter: trace
```

<!-- </snip> -->

ここで最も重要な設定は次のとおりです：

- `level` - 標準的なログレベル。開発環境では通常`debug`または`trace`。本番環境では、慣れ親しんだものを選択してください。
- `pretty_backtrace` - エラーを引き起こしたコード行への明確で簡潔なパスを提供します。開発環境では`true`を使用し、本番環境ではオフにします。本番環境でデバッグが必要な場合は、一時的にオンにして、完了後にオフにすることができます。

### コントローラーのロギング

`server.middlewares`には以下があります：

```yaml
server:
  middlewares:
    #
    # ...
    #
    # Generating a unique request ID and enhancing logging with additional information such as the start and completion of request processing, latency, status code, and other request details.
    logger:
      # Enable/Disable the middleware.
      enable: true
```

詳細なリクエストエラーと、複数のリクエストスコープのエラーを照合するのに役立つ`request-id`を取得するために、これを有効にする必要があります。

### データベース

`database`セクションでライブSQLクエリのロギングオプションがあります：

```yaml
database:
  # When enabled, the sql query will be logged.
  enable_logging: false
```

### エラーの操作

アプリの開発中は主にターミナルでエラーを確認することになります。以下のように表示されます：

```bash
2024-02-xxx DEBUG http-request: tower_http::trace::on_request: started processing request http.method=GET http.uri=/notes http.version=HTTP/1.1 http.user_agent=curl/8.1.2 environment=development request_id=8622e624-9bda-49ce-9730-876f2a8a9a46
2024-02-xxx11T12:19:25.295954Z ERROR http-request: loco_rs::controller: controller_error error.msg=invalid type: string "foo", expected a sequence error.details=JSON(Error("invalid type: string \"foo\", expected a sequence", line: 0, column: 0)) error.chain="" http.method=GET http.uri=/notes http.version=HTTP/1.1 http.user_agent=curl/8.1.2 environment=development request_id=8622e624-9bda-49ce-9730-876f2a8a9a46
```

通常、エラーからは以下を期待できます：

- `error.msg` オペレーター向けのエラーの`to_string()`バージョン。
- `error.detail` 開発者向けのエラーのデバッグ表現。
- エラー**タイプ**（例：`controller_error`）は、言葉によるエラーメッセージよりも検索に適した主要メッセージです。
- エラーは_tracing_イベントとスパンとしてログに記録されるため、カスタムトレーシングサブスクライバーを提供するインフラストラクチャを構築できます。`loco-extras`の[prometheus](https://github.com/loco-rs/loco/blob/master/loco-extras/src/initializers/prometheus.rs)の例を参照してください。

注意事項：

- _エラーチェーン_が実験されましたが、実際にはあまり価値がありません。
- エンドユーザーが目にするエラーは完全に別物です。ユーザーがエラーに対して何もできないことがわかっている場合（例：「データベースオフラインエラー」）、エンドユーザーにはエラーに関する**最小限の内部詳細**を提供するよう努めています。ほとんどの場合、セキュリティ上の理由から、意図的に一般的な「Internal Server Error」になります。

### エラーの生成

コントローラーを構築する際、ハンドラーは`Result<impl IntoResponse>`を返すように記述します。ここでの`Result`はLocoの`Result`であり、Locoの`Error`型も関連付けられています。

Locoの`Error`型を使用する場合、以下のいずれかをレスポンスとして使用できます：

```rust
Err(Error::string("some custom message"));
Err(Error::msg(other_error)); // turns other_error to its string representation
Err(Error::wrap(other_error));
Err(Error::Unauthorized("some message"))

// or through controller helpers:
unauthorized("some message") // create a full response object, calling Err on a created error
```

## イニシャライザー

イニシャライザーは、アプリで必要なインフラストラクチャの「配線」をカプセル化する方法です。イニシャライザーは`src/initializers/`に配置します。

### イニシャライザーの記述

現在、イニシャライザーは`Initializer`トレイトを実装するものであれば何でも構いません：

<!-- <snip id="initializers-trait" inject_from="code" template="rust"> -->

```rust
pub trait Initializer: Sync + Send {
    /// The initializer name or identifier
    fn name(&self) -> String;

    /// Occurs after the app's `before_run`.
    /// Use this to for one-time initializations, load caches, perform web
    /// hooks, etc.
    async fn before_run(&self, _app_context: &AppContext) -> Result<()> {
        Ok(())
    }

    /// Occurs after the app's `after_routes`.
    /// Use this to compose additional functionality and wire it into an Axum
    /// Router
    async fn after_routes(&self, router: AxumRouter, _ctx: &AppContext) -> Result<AxumRouter> {
        Ok(router)
    }
}
```

<!-- </snip> -->

### 例：Axum Sessionの統合

`axum-session`を使用してアプリにセッションを追加したい場合があります。また、その機能を自分のプロジェクト間で共有したり、他の人からそのコードを取得したりしたい場合もあるでしょう。

統合を_イニシャライザー_としてコーディングすれば、この再利用を簡単に実現できます：

```rust
// place this in `src/initializers/axum_session.rs`
#[async_trait]
impl Initializer for AxumSessionInitializer {
    fn name(&self) -> String {
        "axum-session".to_string()
    }

    async fn after_routes(&self, router: AxumRouter, _ctx: &AppContext) -> Result<AxumRouter> {
        let session_config =
            axum_session::SessionConfig::default().with_table_name("sessions_table");
        let session_store =
            axum_session::SessionStore::<axum_session::SessionNullPool>::new(None, session_config)
                .await
                .unwrap();
        let router = router.layer(axum_session::SessionLayer::new(session_store));
        Ok(router)
    }
}
```

そして、アプリの構造は次のようになります：

```
src/
 bin/
 controllers/
    :
    :
 initializers/       <--- a new folder
   mod.rs            <--- a new module
   axum_session.rs   <--- your new initializer
    :
    :
  app.rs   <--- register initializers here
```

### イニシャライザーの使用

独自のイニシャライザーを実装した後、`src/app.rs`で`initializers(..)`フックを実装し、イニシャライザーのVecを提供する必要があります：

<!-- <snip id="app-initializers" inject_from="code" template="rust"> -->

```rust
    async fn initializers(_ctx: &AppContext) -> Result<Vec<Box<dyn Initializer>>> {
        let initializers: Vec<Box<dyn Initializer>> = vec![
            Box::new(initializers::axum_session::AxumSessionInitializer),
            Box::new(initializers::view_engine::ViewEngineInitializer),
            Box::new(initializers::hello_view_engine::HelloViewEngineInitializer),
        ];

        Ok(initializers)
    }
```

<!-- </snip> -->

Locoはアプリの起動プロセス中に、適切な場所でイニシャライザースタックを実行します。

### 他に何ができるか？

現在、イニシャライザーには2つの統合ポイントがあります：

- `before_run` - アプリの実行前に発生します -- これは純粋な「初期化」タイプのフックです。Webフックの送信、メトリックポイントの送信、クリーンアップ、プリフライトチェックなどが可能です。
- `after_routes` - ルートが追加された後に発生します。Axumルーターとその強力なレイヤリング統合ポイントにアクセスできます。ここが最も多くの時間を費やす場所です。

### Railsイニシャライザーとの比較

Railsイニシャライザーは、初期化のために一度だけ実行され、すべてにアクセスできる通常のスクリプトです。「ライブ」Railsアプリにアクセスし、グローバルインスタンスとして変更できることから力を得ています。

Locoでは、Rustではグローバルインスタンスにアクセスして変更することはできません（正当な理由があります！）。そのため、明示的で安全な2つの統合ポイントを提供しています：

1. 純粋な初期化（設定されたアプリへの影響なし）
2. 実行中のアプリとの統合（Axumルーター経由）

Railsイニシャライザーには_順序_と_変更_が必要です。つまり、ユーザーは特定の順序で実行されることを確信できる必要があり（または順序を変更できる）、他の人が以前に設定したイニシャライザーを削除できる必要があります。

Locoでは、ユーザーに_イニシャライザーの完全なVec_を提供させることで、この複雑さを回避しています。Vecは順序付けられており、暗黙的なイニシャライザーはありません。

### グローバルロガーイニシャライザー

一部の開発者はロギングスタックをカスタマイズしたいと考えています。Locoでは、これにはtracingとtracingサブスクライバーの設定が含まれます。

現時点でtracingは再初期化や実行中のtracingスタックの変更を許可していないため、_グローバルtracingスタックを初期化および登録するチャンスは一度だけ_です。

このため、独自のロギングスタック初期化を提供するために使用できる`init_logger`という新しい_アプリレベルフック_を追加しました。

```rust
// in src/app.rs
impl Hooks for App {
    // return `Ok(true)` if you took over initializing logger
    // otherwise, return `Ok(false)` to use the Loco logging stack.
    fn init_logger(_config: &config::Config, _env: &Environment) -> Result<bool> {
        Ok(false)
    }
}
```

独自のロガーを設定した後、初期化を引き継いだことを示すために`Ok(true)`を返します。

## ミドルウェア

`Loco`は[`axum`](https://crates.io/crates/axum)と[`tower`](https://crates.io/crates/tower)の上に構築されたフレームワークです。これらは、ルートとハンドラーにミドルウェアとして[レイヤー](https://docs.rs/tower/latest/tower/trait.Layer.html)と[サービス](https://docs.rs/tower/latest/tower/trait.Service.html)を追加する方法を提供します。

ミドルウェアは、リクエストに前処理と後処理を追加する方法です。これは、ロギング、認証、レート制限、ルート固有の処理などに使用できます。

### ソースコード

`Loco`のルートミドルウェア/レイヤーの実装は`axum`の[`Router::layer`](https://github.com/tokio-rs/axum/blob/main/axum/src/routing/mod.rs#L275)に似ています。ミドルウェアのソースコードは[`src/controllers/routes`](https://github.com/loco-rs/loco/blob/master/src/controller/routes.rs)ディレクトリにあります。この`layer`関数は、ルートの各ハンドラーにミドルウェアレイヤーをアタッチします。

```rust
// src/controller/routes.rs
use axum::{extract::Request, response::IntoResponse, routing::Route};
use tower::{Layer, Service};

impl Routes {
    pub fn layer<L>(self, layer: L) -> Self
        where
            L: Layer<Route> + Clone + Send + 'static,
            L::Service: Service<Request> + Clone + Send + 'static,
            <L::Service as Service<Request>>::Response: IntoResponse + 'static,
            <L::Service as Service<Request>>::Error: Into<Infallible> + 'static,
            <L::Service as Service<Request>>::Future: Send + 'static,
    {
        Self {
            prefix: self.prefix,
            handlers: self
                .handlers
                .iter()
                .map(|handler| Handler {
                    uri: handler.uri.clone(),
                    actions: handler.actions.clone(),
                    method: handler.method.clone().layer(layer.clone()),
                })
                .collect(),
        }
    }
}
```

### Basic Middleware

In this example, we will create a basic middleware that will log the request method and path.

```rust
// src/controllers/middleware/log.rs
use std::{
    convert::Infallible,
    task::{Context, Poll},
};

use axum::{
    body::Body,
    extract::{FromRequestParts, Request},
    response::Response,
};
use futures_util::future::BoxFuture;
use loco_rs::prelude::{auth::JWTWithUser, *};
use tower::{Layer, Service};

use crate::models::{users};

#[derive(Clone)]
pub struct LogLayer;

impl LogLayer {
    pub fn new() -> Self {
        Self {}
    }
}

impl<S> Layer<S> for LogLayer {
    type Service = LogService<S>;

    fn layer(&self, inner: S) -> Self::Service {
        Self::Service {
            inner,
        }
    }
}

#[derive(Clone)]
pub struct LogService<S> {
    // S is the inner service, in the case, it is the `/auth/register` handler
    inner: S,
}

/// Implement the Service trait for LogService
/// # Generics
/// * `S` - The inner service, in this case is the `/auth/register` handler
/// * `B` - The body type
impl<S, B> Service<Request<B>> for LogService<S>
    where
        S: Service<Request<B>, Response=Response<Body>, Error=Infallible> + Clone + Send + 'static, /* Inner Service must return Response<Body> and never error, which is typical for handlers */
        S::Future: Send + 'static,
        B: Send + 'static,
{
    // Response type is the same as the inner service / handler
    type Response = S::Response;
    // Error type is the same as the inner service / handler
    type Error = S::Error;
    // Future type is the same as the inner service / handler
    type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>;

    // poll_ready is used to check if the service is ready to process a request
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        // Our middleware doesn't care about backpressure, so it's ready as long
        // as the inner service is ready.
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: Request<B>) -> Self::Future {
        let clone = self.inner.clone();
        // take the service that was ready
        let mut inner = std::mem::replace(&mut self.inner, clone);
        Box::pin(async move {
            let (mut parts, body) = req.into_parts();
            tracing::info!("Request: {:?} {:?}", parts.method, parts.uri.path());
            let req = Request::from_parts(parts, body);
            inner.call(req).await
        })
    }
}
```

At the first glance, this middleware is a bit overwhelming. Let's break it down.

The `LogLayer` is a [`tower::Layer`](https://docs.rs/tower/latest/tower/trait.Layer.html) that wraps around the inner
service.

The `LogService` is a [`tower::Service`](https://docs.rs/tower/latest/tower/trait.Service.html) that implements
the `Service` trait for the request.

### Generics Explanation

**`Layer`**

In the `Layer` trait, `S` represents the inner service, which in this case is the `/auth/register` handler. The `layer`
function takes this inner service and returns a new service that wraps around it.

**`Service`**

`S` is the inner service, in this case, it is the `/auth/register` handler. If we have a look about
the [`get`](https://docs.rs/axum/latest/axum/routing/method_routing/fn.get.html), [`post`](https://docs.rs/axum/latest/axum/routing/method_routing/fn.post.html), [`put`](https://docs.rs/axum/latest/axum/routing/method_routing/fn.put.html), [`delete`](https://docs.rs/axum/latest/axum/routing/method_routing/fn.delete.html)
functions which we use for handlers, they all return
a [`MethodRoute<S, Infallible>`(Which is a service)](https://docs.rs/axum/latest/axum/routing/method_routing/struct.MethodRouter.html).

Therefore, `S: Service<Request<B>, Response = Response<Body>, Error = Infallible>` means it takes in a `Request<B>`(
Request with a body) and returns a `Response<Body>`. The `Error` is `Infallible` which means the handler never errors.

`S::Future: Send + 'static` means the future of the inner service must implement `Send` trait and `'static`.

`type Response = S::Response` means the response type of the middleware is the same as the inner service.

`type Error = S::Error` means the error type of the middleware is the same as the inner service.

`type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>` means the future type of the middleware is the
same as the inner service.

`B: Send + 'static` means the request body type must implement the `Send` trait and `'static`.

### Function Explanation

**`LogLayer`**

The `LogLayer::new` function is used to create a new instance of the `LogLayer`.

**`LogService`**

The `LogService::poll_ready` function is used to check if the service is ready to process a request. It can be used for
backpressure, for more information see
the [`tower::Service` documentation](https://docs.rs/tower/latest/tower/trait.Service.html)
and [Tokio tutorial](https://tokio.rs/blog/2021-05-14-inventing-the-service-trait#backpressure).

The `LogService::call` function is used to process the request. In this case, we are logging the request method and
path. Then we are calling the inner service with the request.

**Importance of `poll_ready`:**

In the Tower framework, before a service can be used to handle a request, it must be
checked for readiness
using the
`poll_ready` method. This method returns `Poll::Ready(Ok(()))` when the service is ready to process a request. If a
service is not ready, it may return `Poll::Pending`, indicating that the caller should wait before sending a request.
This mechanism ensures that the service has the necessary resources or state to process the request efficiently and
correctly.

**Cloning and Readiness**

When cloning a service, particularly to move it into a boxed future or similar context, it's crucial to understand that
the clone does not inherit the readiness state of the original service. Each clone of a service maintains its own state.
This means that even if the original service was ready `(Poll::Ready(Ok(())))`, the cloned service might not be in the
same state immediately after cloning. This can lead to issues where a cloned service is used before it is ready,
potentially causing panics or other failures.

**Correct approach to cloning services using `std::mem::replace`**
To handle cloning correctly, it's recommended to use `std::mem::replace` to swap the ready service with its clone in a
controlled manner. This approach ensures that the service being used to handle the request is the one that has been
verified as ready. Here's how it works:

- Clone the service: First, create a clone of the service. This clone will eventually replace the original service in
  the service handler.
- Replace the original with the clone: Use `std::mem::replace` to swap the original service with the clone. This
  operation ensures that the service handler continues to hold a service instance.
- Use the original service to handle the request: Since the original service was already checked for readiness (via
  `poll_ready`), it's safe to use it to handle the incoming request. The clone, now in the handler, will be the one
  checked for readiness next time.

This method ensures that each service instance used to handle requests is always the one that has been explicitly
checked for readiness, thus maintaining the integrity and reliability of the service handling process.

Here is a simplified example to illustrate this pattern:

```rust
// Wrong
fn call(&mut self, req: Request<B>) -> Self::Future {
    let mut inner = self.inner.clone();
    Box::pin(async move {
        /* ... */
        inner.call(req).await
    })
}

// Correct
fn call(&mut self, req: Request<B>) -> Self::Future {
    let clone = self.inner.clone();
    // take the service that was ready
    let mut inner = std::mem::replace(&mut self.inner, clone);
    Box::pin(async move {
        /* ... */
        inner.call(req).await
    })
}
```

In this example, `inner` is the service that was ready, and after handling the request, `self.inner` now holds the
clone, which will be checked for readiness in the next cycle. This careful management of service readiness and cloning
is essential for maintaining robust and error-free service operations in asynchronous Rust applications using Tower.

[Tower Service Cloning Documentation](https://docs.rs/tower/latest/tower/trait.Service.html#be-careful-when-cloning-inner-services)

### Adding Middleware to Handler

Add the middleware to the `auth::register` handler.

```rust
// src/controllers/auth.rs
pub fn routes() -> Routes {
    Routes::new()
        .prefix("auth")
        .add("/register", post(register).layer(middlewares::log::LogLayer::new()))
}
```

Now when you make a request to the `auth::register` handler, you will see the request method and path logged.

```shell
2024-XX-XXTXX:XX:XX.XXXXXZ  INFO http-request: xx::controllers::middleware::log Request: POST "/auth/register" http.method=POST http.uri=/auth/register http.version=HTTP/1.1  environment=development request_id=xxxxx
```

## Adding Middleware to Route

Add the middleware to the `auth` route.

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

Now when you make a request to any handler in the `auth` route, you will see the request method and path logged.

```shell
2024-XX-XXTXX:XX:XX.XXXXXZ  INFO http-request: xx::controllers::middleware::log Request: POST "/auth/register" http.method=POST http.uri=/auth/register http.version=HTTP/1.1  environment=development request_id=xxxxx
```

### Advanced Middleware (With AppContext)

There will be times when you need to access the `AppContext` in your middleware. For example, you might want to access
the database connection to perform some authorization checks. To do this, you can add the `AppContext` to
the `Layer` and `Service`.

Here we will create a middleware that checks the JWT token and gets the user from the database then prints the user's
name

```rust
// src/controllers/middleware/log.rs
use std::{
    convert::Infallible,
    task::{Context, Poll},
};

use axum::{
    body::Body,
    extract::{FromRequestParts, Request},
    response::Response,
};
use futures_util::future::BoxFuture;
use loco_rs::prelude::{auth::JWTWithUser, *};
use tower::{Layer, Service};

use crate::models::{users};

#[derive(Clone)]
pub struct LogLayer {
    state: AppContext,
}

impl LogLayer {
    pub fn new(state: AppContext) -> Self {
        Self { state }
    }
}

impl<S> Layer<S> for LogLayer {
    type Service = LogService<S>;

    fn layer(&self, inner: S) -> Self::Service {
        Self::Service {
            inner,
            state: self.state.clone(),
        }
    }
}

#[derive(Clone)]
pub struct LogService<S> {
    inner: S,
    state: AppContext,
}

impl<S, B> Service<Request<B>> for LogService<S>
    where
        S: Service<Request<B>, Response=Response<Body>, Error=Infallible> + Clone + Send + 'static, /* Inner Service must return Response<Body> and never error */
        S::Future: Send + 'static,
        B: Send + 'static,
{
    // Response type is the same as the inner service / handler
    type Response = S::Response;
    // Error type is the same as the inner service / handler
    type Error = S::Error;
    // Future type is the same as the inner service / handler
    type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>;
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: Request<B>) -> Self::Future {
        let state = self.state.clone();
        let clone = self.inner.clone();
        // take the service that was ready
        let mut inner = std::mem::replace(&mut self.inner, clone);
        Box::pin(async move {
            // Example of extracting JWT token from the request
            let (mut parts, body) = req.into_parts();
            let auth = JWTWithUser::<users::Model>::from_request_parts(&mut parts, &state).await;

            match auth {
                Ok(auth) => {
                    // Example of getting user from the database
                    let user = users::Model::find_by_email(&state.db, &auth.user.email).await.unwrap();
                    tracing::info!("User: {}", user.name);
                    let req = Request::from_parts(parts, body);
                    inner.call(req).await
                }
                Err(_) => {
                    // Handle error, e.g., return an unauthorized response
                    Ok(Response::builder()
                        .status(401)
                        .body(Body::empty())
                        .unwrap()
                        .into_response())
                }
            }
        })
    }
}
```

In this example, we have added the `AppContext` to the `LogLayer` and `LogService`. We are using the `AppContext` to get
the database connection and the JWT token for pre-processing.

### Adding Middleware to Route (advanced)

Add the middleware to the `notes` route.

```rust
// src/app.rs
pub struct App;

#[async_trait]
impl Hooks for App {
    fn routes(ctx: &AppContext) -> AppRoutes {
        AppRoutes::with_default_routes()
            .add_route(controllers::notes::routes().layer(middlewares::log::LogLayer::new(ctx)))
    }
}
```

Now when you make a request to any handler in the `notes` route, you will see the user's name logged.

```shell
2024-XX-XXTXX:XX:XX.XXXXXZ  INFO http-request: xx::controllers::middleware::log User: John Doe  environment=development request_id=xxxxx
```

### Adding Middleware to Handler (advanced)

In order to add the middleware to the handler, you need to add the `AppContext` to the `routes` function
in `src/app.rs`.

```rust
// src/app.rs
pub struct App;

#[async_trait]
impl Hooks for App {
    fn routes(ctx: &AppContext) -> AppRoutes {
        AppRoutes::with_default_routes()
            .add_route(
                controllers::notes::routes(ctx)
            )
    }
}
```

Then add the middleware to the `notes::create` handler.

```rust
// src/controllers/notes.rs
pub fn routes(ctx: &AppContext) -> Routes {
    Routes::new()
        .prefix("notes")
        .add("/create", post(create).layer(middlewares::log::LogLayer::new(ctx)))
}
```

Now when you make a request to the `notes::create` handler, you will see the user's name logged.

```shell
2024-XX-XXTXX:XX:XX.XXXXXZ  INFO http-request: xx::controllers::middleware::log User: John Doe  environment=development request_id=xxxxx
```

## Application SharedStore

Loco provides a flexible mechanism called `SharedStore` within the `AppContext` to store and share arbitrary custom data or services across your application. This feature allows you to inject your own types into the application context without modifying Loco's core structures, enhancing pluggability and customization.

`AppContext.shared_store` is a type-safe, thread-safe heterogeneous storage. You can store any type that implements `'static + Send + Sync`.

### Why Use SharedStore?

- **Sharing Custom Services:** Inject your own service clients (e.g., a custom API client) and access them from controllers or background workers.
- **Storing Configuration:** Keep application-specific configuration objects accessible globally.
- **Shared State:** Manage state needed by different parts of your application.

### How to Use SharedStore

You typically insert your custom data into the `shared_store` during application startup (e.g., in `src/app.rs`) and then retrieve it within your controllers or other components.

**1. Define Your Data Structures:**

Create the structs for the data or services you want to share. Note whether they implement `Clone`.

```rust
// In src/app.rs or a dedicated module (e.g., src/services.rs)

// This service can be cloned
#[derive(Clone, Debug)]
pub struct MyClonableService {
    pub api_key: String,
}

// This service cannot (or should not) be cloned
#[derive(Debug)]
pub struct MyNonClonableService {
    pub api_key: String,
}
```

**2. Insert into SharedStore (in `src/app.rs`):**

A good place to insert your shared data is the `after_context` hook in your `App`'s `Hooks` implementation.

```rust
// In src/app.rs

use crate::MyClonableService; // Import your structs
use crate::MyNonClonableService;

pub struct App;
#[async_trait]
impl Hooks for App {
    // ... other Hooks methods (app_name, boot, etc.) ...

    async fn after_context(mut ctx: AppContext) -> Result<AppContext> {
        // Create instances of your services/data
        let clonable_service = MyClonableService {
            api_key: "key-cloned-12345".to_string(),
        };
        let non_clonable_service = MyNonClonableService {
            api_key: "key-ref-67890".to_string(),
        };

        // Insert them into the shared store
        ctx.shared_store.insert(clonable_service);
        ctx.shared_store.insert(non_clonable_service);

        Ok(ctx)
    }

    // ... rest of Hooks implementation ...
}
```

**3. Retrieve from SharedStore (in Controllers):**

You have two main ways to retrieve data in your controllers:

- **Using the `SharedStore(var)` Extractor (for `Clone`-able types):**
  This is the most convenient way if your type implements `Clone`. The extractor retrieves and _clones_ the data for you.

  ```rust
  // In src/controllers/some_controller.rs
  use loco_rs::prelude::*;
  use crate::app::MyClonableService; // Or wherever it's defined

  #[axum::debug_handler]
  pub async fn index(
      // Extracts and clones MyClonableService into `service`
      SharedStore(service): SharedStore<MyClonableService>,
  ) -> impl IntoResponse {
      tracing::info!("Using Cloned Service API Key: {}", service.api_key);
      format::empty()
  }
  ```

- **Using `ctx.shared_store.get_ref()` (for Non-`Clone`-able types or avoiding clones):**
  Use this method when your type doesn't implement `Clone` or when you want to avoid the performance cost of cloning. It gives you a reference (`RefGuard<T>`) to the data.

  ```rust
  // In src/controllers/some_controller.rs
  use loco_rs::prelude::*;
  use crate::app::MyNonClonableService; // Or wherever it's defined

  #[axum::debug_handler]
  pub async fn index(
      State(ctx): State<AppContext>, // Need the AppContext state
  ) -> Result<impl IntoResponse> {
      // Get a reference to the non-clonable service
      let service_ref = ctx.shared_store.get_ref::<MyNonClonableService>()
          .ok_or_else(|| {
              tracing::error!("MyNonClonableService not found in shared store");
              Error::InternalServerError // Or a more specific error
          })?;

      // Access fields via the reference guard
      tracing::info!("Using Non-Cloned Service API Key: {}", service_ref.api_key);
      format::empty()
  }
  ```

**Summary:**

- Use `SharedStore` in `AppContext` to share custom services or data.
- Insert data during app setup (e.g., `after_context` in `src/app.rs`).
- Use the `SharedStore(var)` extractor for convenient access to `Clone`-able types (clones the data).
- Use `ctx.shared_store.get_ref::<T>()` to get a reference to non-`Clone`-able types or to avoid cloning for performance reasons.
