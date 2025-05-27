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
# アプリケーションのロギング設定
logger:
  # ロギングの有効化または無効化
  enable: true
  # pretty backtraceを有効化（RUST_BACKTRACE=1に設定）
  pretty_backtrace: true
  # ログレベル、オプション：trace、debug、info、warn、error
  level: debug
  # ロギング形式の定義。オプション：compact、pretty、json
  format: compact
  # デフォルトでは、ロガーはあなたのコードまたは`loco`フレームワークからのログのみをフィルタリングします。すべてのサードパーティライブラリを表示するには
  # 以下の行をコメント解除して、すべてのサードパーティライブラリを表示するためにこの設定を有効にし、ロガーフィルターをオーバーライドできます。
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
    # 一意のリクエストIDを生成し、リクエスト処理の開始と完了、レイテンシ、ステータスコード、その他のリクエスト詳細などの追加情報でロギングを強化します。
    logger:
      # ミドルウェアの有効化/無効化
      enable: true
```

詳細なリクエストエラーと、複数のリクエストスコープのエラーを照合するのに役立つ`request-id`を取得するために、これを有効にする必要があります。

### データベース

`database`セクションでライブSQLクエリのロギングオプションがあります：

```yaml
database:
  # 有効にすると、SQLクエリがログに記録されます。
  enable_logging: false
```

### エラーの扱い方

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

### エラーの作成

コントローラーを構築する際、ハンドラーは`Result<impl IntoResponse>`を返すように記述します。ここでの`Result`はLocoの`Result`であり、Locoの`Error`型も関連付けられています。

Locoの`Error`型を使用する場合、以下のいずれかをレスポンスとして使用できます：

```rust
Err(Error::string("カスタムメッセージ"));
Err(Error::msg(other_error)); // other_errorをその文字列表現に変換
Err(Error::wrap(other_error));
Err(Error::Unauthorized("メッセージ"))

// またはコントローラーヘルパー経由で：
unauthorized("メッセージ") // 完全なレスポンスオブジェクトを作成し、作成されたエラーに対してErrを呼び出す
```

## イニシャライザー

イニシャライザーは、アプリで必要なインフラストラクチャの「配線」をカプセル化する方法です。イニシャライザーは`src/initializers/`に配置します。

### イニシャライザーの記述

現在、イニシャライザーは`Initializer`トレイトを実装するものであれば何でも構いません：

<!-- <snip id="initializers-trait" inject_from="code" template="rust"> -->

```rust
pub trait Initializer: Sync + Send {
    /// イニシャライザーの名前または識別子
    fn name(&self) -> String;

    /// アプリの`before_run`の後に発生します。
    /// これを使用して、一度だけの初期化、キャッシュの読み込み、Webフックの実行などを行います。
    async fn before_run(&self, _app_context: &AppContext) -> Result<()> {
        Ok(())
    }

    /// アプリの`after_routes`の後に発生します。
    /// これを使用して、追加の機能を構成し、Axumルーターに配線します
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
// これを`src/initializers/axum_session.rs`に配置します
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
 initializers/       <--- 新しいフォルダ
   mod.rs            <--- 新しいモジュール
   axum_session.rs   <--- 新しいイニシャライザー
    :
    :
  app.rs   <--- ここでイニシャライザーを登録
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
// src/app.rs内
impl Hooks for App {
    // ロガーの初期化を引き継いだ場合は`Ok(true)`を返す
    // それ以外の場合は、Locoのロギングスタックを使用するために`Ok(false)`を返す。
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

### 基本的なミドルウェア

この例では、リクエストメソッドとパスをログに記録する基本的なミドルウェアを作成します。

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

一見すると、このミドルウェアは少し圧倒的かもしれません。分解してみましょう。

`LogLayer`は内部サービスをラップする[`tower::Layer`](https://docs.rs/tower/latest/tower/trait.Layer.html)です。

`LogService`はリクエストに対して`Service`トレイトを実装する[`tower::Service`](https://docs.rs/tower/latest/tower/trait.Service.html)です。

### ジェネリクスの説明

**`Layer`**

`Layer`トレイトでは、`S`は内部サービスを表し、この場合は`/auth/register`ハンドラーです。`layer`関数はこの内部サービスを受け取り、それをラップする新しいサービスを返します。

**`Service`**

`S`は内部サービスで、この場合は`/auth/register`ハンドラーです。ハンドラーに使用する[`get`](https://docs.rs/axum/latest/axum/routing/method_routing/fn.get.html)、[`post`](https://docs.rs/axum/latest/axum/routing/method_routing/fn.post.html)、[`put`](https://docs.rs/axum/latest/axum/routing/method_routing/fn.put.html)、[`delete`](https://docs.rs/axum/latest/axum/routing/method_routing/fn.delete.html)関数を見ると、これらはすべて[`MethodRoute<S, Infallible>`（これはサービスです）](https://docs.rs/axum/latest/axum/routing/method_routing/struct.MethodRouter.html)を返します。

したがって、`S: Service<Request<B>, Response = Response<Body>, Error = Infallible>`は、`Request<B>`（ボディを持つリクエスト）を受け取り、`Response<Body>`を返すことを意味します。`Error`は`Infallible`で、ハンドラーがエラーを発生させないことを意味します。

`S::Future: Send + 'static`は、内部サービスのフューチャーが`Send`トレイトと`'static`を実装する必要があることを意味します。

`type Response = S::Response`は、ミドルウェアのレスポンス型が内部サービスと同じであることを意味します。

`type Error = S::Error`は、ミドルウェアのエラー型が内部サービスと同じであることを意味します。

`type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>`は、ミドルウェアのフューチャー型が内部サービスと同じであることを意味します。

`B: Send + 'static`は、リクエストボディ型が`Send`トレイトと`'static`を実装する必要があることを意味します。

### 関数の説明

**`LogLayer`**

`LogLayer::new`関数は、`LogLayer`の新しいインスタンスを作成するために使用されます。

**`LogService`**

`LogService::poll_ready`関数は、サービスがリクエストを処理する準備ができているかどうかを確認するために使用されます。これはバックプレッシャーに使用できます。詳細については、[`tower::Service`のドキュメント](https://docs.rs/tower/latest/tower/trait.Service.html)と[Tokioチュートリアル](https://tokio.rs/blog/2021-05-14-inventing-the-service-trait#backpressure)を参照してください。

`LogService::call`関数は、リクエストを処理するために使用されます。この場合、リクエストメソッドとパスをログに記録し、次に内部サービスをリクエストと共に呼び出しています。

**`poll_ready`の重要性：**

Towerフレームワークでは、サービスがリクエストを処理するために使用される前に、`poll_ready`メソッドを使用して準備状況を確認する必要があります。このメソッドは、サービスがリクエストを処理する準備ができているときに`Poll::Ready(Ok(()))`を返します。サービスの準備ができていない場合、`Poll::Pending`を返すことがあり、呼び出し元はリクエストを送信する前に待機する必要があることを示します。このメカニズムにより、サービスがリクエストを効率的かつ正確に処理するために必要なリソースまたは状態を持っていることが保証されます。

**クローニングと準備状況**

サービスをクローンする場合、特にボックス化されたフューチャーや同様のコンテキストに移動する場合、クローンが元のサービスの準備状況を継承しないことを理解することが重要です。サービスの各クローンは独自の状態を維持します。これは、元のサービスが準備完了（`Poll::Ready(Ok(()))`）であっても、クローンされたサービスがクローン直後に同じ状態にない可能性があることを意味します。これは、準備ができる前にクローンされたサービスが使用される問題につながり、パニックやその他の障害を引き起こす可能性があります。

**`std::mem::replace`を使用したサービスのクローニングの正しいアプローチ**
クローニングを正しく処理するには、準備完了のサービスをそのクローンと制御された方法で交換するために`std::mem::replace`を使用することをお勧めします。このアプローチにより、リクエストを処理するために使用されるサービスが、準備完了として確認されたものであることが保証されます。動作は次のとおりです：

- サービスをクローンする：まず、サービスのクローンを作成します。このクローンは最終的にサービスハンドラー内の元のサービスを置き換えます。
- 元のサービスをクローンと置き換える：`std::mem::replace`を使用して、元のサービスをクローンと交換します。この操作により、サービスハンドラーが引き続きサービスインスタンスを保持することが保証されます。
- 元のサービスを使用してリクエストを処理する：元のサービスは既に準備状況が確認されている（`poll_ready`経由）ため、着信リクエストを処理するために安全に使用できます。ハンドラー内のクローンは、次回準備状況が確認されるものになります。

この方法により、リクエストを処理するために使用される各サービスインスタンスが、常に明示的に準備状況が確認されたものであることが保証され、サービス処理プロセスの整合性と信頼性が維持されます。

このパターンを説明する簡単な例を以下に示します：

```rust
// 間違い
fn call(&mut self, req: Request<B>) -> Self::Future {
    let mut inner = self.inner.clone();
    Box::pin(async move {
        /* ... */
        inner.call(req).await
    })
}

// 正しい
fn call(&mut self, req: Request<B>) -> Self::Future {
    let clone = self.inner.clone();
    // 準備ができていたサービスを取得
    let mut inner = std::mem::replace(&mut self.inner, clone);
    Box::pin(async move {
        /* ... */
        inner.call(req).await
    })
}
```

この例では、`inner`は準備ができていたサービスであり、リクエストを処理した後、`self.inner`はクローンを保持し、次のサイクルで準備状況が確認されます。このサービスの準備状況とクローニングの慎重な管理は、Towerを使用する非同期Rustアプリケーションで堅牢でエラーのないサービス操作を維持するために不可欠です。

[Towerサービスクローニングのドキュメント](https://docs.rs/tower/latest/tower/trait.Service.html#be-careful-when-cloning-inner-services)

### ハンドラーへのミドルウェアの追加

`auth::register`ハンドラーにミドルウェアを追加します。

```rust
// src/controllers/auth.rs
pub fn routes() -> Routes {
    Routes::new()
        .prefix("auth")
        .add("/register", post(register).layer(middlewares::log::LogLayer::new()))
}
```

これで`auth::register`ハンドラーにリクエストを送信すると、リクエストメソッドとパスがログに記録されます。

```shell
2024-XX-XXTXX:XX:XX.XXXXXZ  INFO http-request: xx::controllers::middleware::log Request: POST "/auth/register" http.method=POST http.uri=/auth/register http.version=HTTP/1.1  environment=development request_id=xxxxx
```

## ルートへのミドルウェアの追加

`auth`ルートにミドルウェアを追加します。

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

これで`auth`ルート内の任意のハンドラーにリクエストを送信すると、リクエストメソッドとパスがログに記録されます。

```shell
2024-XX-XXTXX:XX:XX.XXXXXZ  INFO http-request: xx::controllers::middleware::log Request: POST "/auth/register" http.method=POST http.uri=/auth/register http.version=HTTP/1.1  environment=development request_id=xxxxx
```

### 高度なミドルウェア（AppContextを使用）

ミドルウェア内で`AppContext`にアクセスする必要がある場合があります。例えば、いくつかの認証チェックを実行するためにデータベース接続にアクセスしたい場合があります。これを行うには、`Layer`と`Service`に`AppContext`を追加できます。

ここでは、JWTトークンをチェックし、データベースからユーザーを取得してユーザー名を出力するミドルウェアを作成します

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
        S: Service<Request<B>, Response=Response<Body>, Error=Infallible> + Clone + Send + 'static, /* 内部サービスはResponse<Body>を返し、エラーを発生させない必要があります */
        S::Future: Send + 'static,
        B: Send + 'static,
{
    // レスポンス型は内部サービス/ハンドラーと同じ
    type Response = S::Response;
    // エラー型は内部サービス/ハンドラーと同じ
    type Error = S::Error;
    // フューチャー型は内部サービス/ハンドラーと同じ
    type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>;
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: Request<B>) -> Self::Future {
        let state = self.state.clone();
        let clone = self.inner.clone();
        // 準備ができていたサービスを取得
        let mut inner = std::mem::replace(&mut self.inner, clone);
        Box::pin(async move {
            // リクエストからJWTトークンを抽出する例
            let (mut parts, body) = req.into_parts();
            let auth = JWTWithUser::<users::Model>::from_request_parts(&mut parts, &state).await;

            match auth {
                Ok(auth) => {
                    // データベースからユーザーを取得する例
                    let user = users::Model::find_by_email(&state.db, &auth.user.email).await.unwrap();
                    tracing::info!("ユーザー: {}", user.name);
                    let req = Request::from_parts(parts, body);
                    inner.call(req).await
                }
                Err(_) => {
                    // エラーの処理、例：未認証レスポンスを返す
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

この例では、`LogLayer`と`LogService`に`AppContext`を追加しました。前処理のためにデータベース接続とJWTトークンを取得するために`AppContext`を使用しています。

### ルートへのミドルウェアの追加（高度）

`notes`ルートにミドルウェアを追加します。

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

これで`notes`ルート内の任意のハンドラーにリクエストを送信すると、ユーザー名がログに記録されます。

```shell
2024-XX-XXTXX:XX:XX.XXXXXZ  INFO http-request: xx::controllers::middleware::log ユーザー: John Doe  environment=development request_id=xxxxx
```

### ハンドラーへのミドルウェアの追加（高度）

ハンドラーにミドルウェアを追加するには、`src/app.rs`の`routes`関数に`AppContext`を追加する必要があります。

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

次に、`notes::create`ハンドラーにミドルウェアを追加します。

```rust
// src/controllers/notes.rs
pub fn routes(ctx: &AppContext) -> Routes {
    Routes::new()
        .prefix("notes")
        .add("/create", post(create).layer(middlewares::log::LogLayer::new(ctx)))
}
```

これで`notes::create`ハンドラーにリクエストを送信すると、ユーザー名がログに記録されます。

```shell
2024-XX-XXTXX:XX:XX.XXXXXZ  INFO http-request: xx::controllers::middleware::log ユーザー: John Doe  environment=development request_id=xxxxx
```

## アプリケーションSharedStore

Locoは、アプリケーション全体で任意のカスタムデータやサービスを保存および共有するための`SharedStore`と呼ばれる柔軟なメカニズムを`AppContext`内に提供します。この機能により、Locoのコア構造を変更することなく、独自の型をアプリケーションコンテキストに注入でき、プラグイン性とカスタマイズが向上します。

`AppContext.shared_store`は、タイプセーフでスレッドセーフな異種ストレージです。`'static + Send + Sync`を実装する任意の型を保存できます。

### SharedStoreを使用する理由

- **カスタムサービスの共有：** 独自のサービスクライアント（例：カスタムAPIクライアント）を注入し、コントローラーやバックグラウンドワーカーからアクセスします。
- **設定の保存：** アプリケーション固有の設定オブジェクトをグローバルにアクセス可能な状態で保持します。
- **共有状態：** アプリケーションの異なる部分で必要な状態を管理します。

### SharedStoreの使用方法

通常、アプリケーションの起動時（例：`src/app.rs`内）にカスタムデータを`shared_store`に挿入し、コントローラーやその他のコンポーネント内で取得します。

**1. データ構造の定義：**

共有したいデータやサービスの構造体を作成します。`Clone`を実装しているかどうかに注意してください。

```rust
// src/app.rs内、または専用のモジュール（例：src/services.rs）内

// このサービスはクローン可能
#[derive(Clone, Debug)]
pub struct MyClonableService {
    pub api_key: String,
}

// このサービスはクローンできない（またはすべきでない）
#[derive(Debug)]
pub struct MyNonClonableService {
    pub api_key: String,
}
```

**2. SharedStoreへの挿入（`src/app.rs`内）：**

共有データを挿入する良い場所は、`App`の`Hooks`実装内の`after_context`フックです。

```rust
// src/app.rs内

use crate::MyClonableService; // 構造体をインポート
use crate::MyNonClonableService;

pub struct App;
#[async_trait]
impl Hooks for App {
    // ... その他のHooksメソッド（app_name、bootなど）...

    async fn after_context(mut ctx: AppContext) -> Result<AppContext> {
        // サービス/データのインスタンスを作成
        let clonable_service = MyClonableService {
            api_key: "key-cloned-12345".to_string(),
        };
        let non_clonable_service = MyNonClonableService {
            api_key: "key-ref-67890".to_string(),
        };

        // 共有ストアに挿入
        ctx.shared_store.insert(clonable_service);
        ctx.shared_store.insert(non_clonable_service);

        Ok(ctx)
    }

    // ... Hooks実装の残りの部分...
}
```

**3. SharedStoreからの取得（コントローラー内）：**

コントローラー内でデータを取得する主な方法は2つあります：

- **`SharedStore(var)`エクストラクターの使用（`Clone`可能な型の場合）：**
  型が`Clone`を実装している場合、これが最も便利な方法です。エクストラクターがデータを取得して_クローン_します。

  ```rust
  // src/controllers/some_controller.rs内
  use loco_rs::prelude::*;
  use crate::app::MyClonableService; // または定義された場所

  #[axum::debug_handler]
  pub async fn index(
      // MyClonableServiceを`service`に抽出してクローン
      SharedStore(service): SharedStore<MyClonableService>,
  ) -> impl IntoResponse {
      tracing::info!("クローンされたサービスAPIキーを使用: {}", service.api_key);
      format::empty()
  }
  ```

- **`ctx.shared_store.get_ref()`の使用（`Clone`不可能な型またはクローンを避ける場合）：**
  型が`Clone`を実装していない場合、またはクローンのパフォーマンスコストを避けたい場合にこのメソッドを使用します。データへの参照（`RefGuard<T>`）を提供します。

  ```rust
  // src/controllers/some_controller.rs内
  use loco_rs::prelude::*;
  use crate::app::MyNonClonableService; // または定義された場所

  #[axum::debug_handler]
  pub async fn index(
      State(ctx): State<AppContext>, // AppContext状態が必要
  ) -> Result<impl IntoResponse> {
      // クローン不可能なサービスへの参照を取得
      let service_ref = ctx.shared_store.get_ref::<MyNonClonableService>()
          .ok_or_else(|| {
              tracing::error!("共有ストアにMyNonClonableServiceが見つかりません");
              Error::InternalServerError // またはより具体的なエラー
          })?;

      // 参照ガード経由でフィールドにアクセス
      tracing::info!("クローンされていないサービスAPIキーを使用: {}", service_ref.api_key);
      format::empty()
  }
  ```

**まとめ：**

- カスタムサービスやデータを共有するために`AppContext`の`SharedStore`を使用します。
- アプリのセットアップ中（例：`src/app.rs`の`after_context`）にデータを挿入します。
- `Clone`可能な型への便利なアクセスのために`SharedStore(var)`エクストラクターを使用します（データをクローンします）。
- `Clone`不可能な型への参照を取得する場合、またはパフォーマンス上の理由でクローンを避けたい場合は`ctx.shared_store.get_ref::<T>()`を使用します。
