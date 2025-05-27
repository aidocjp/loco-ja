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

アプリがプロキシやロードバランサー（Nginx、ELBなど）の背後にある場合、インターネットに直接面していません。そのため、接続しているクライアントIPを調べようとすると、実際にはロードバランサーのIPを示すソケットを取得することになります。

ロードバランサーまたはプロキシは、実際のクライアントIPに対してソケット作業を行い、プロキシバック接続を介してアプリに負荷を与える責任があります。

そのため、アプリが実際のクライアントIPを取得する具体的なビジネスニーズがある場合、プロキシとロードバランサーがこの情報を提供するために使用するデファクトスタンダードである`X-Forwarded-For`ヘッダーを使用する必要があります。

Locoは`RemoteIP`ミドルウェアを設定するための`remote_ip`セクションを提供します：

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

次に、`RemoteIP`エクストラクターを使用してIPを取得します：

```rust
#[debug_handler]
pub async fn list(ip: RemoteIP, State(ctx): State<AppContext>) -> Result<Response> {
    println!("remote ip {ip}");
    format::json(Entity::find().all(&ctx.db).await?)
}
```

`RemoteIP`ミドルウェアを使用する際は、現在のアーキテクチャに対するセキュリティへの影響に注意してください（ドキュメントと設定セクションに記載されています）：アプリがプロキシの背後にない場合、誰でもヘッダーを任意の値に設定でき、特に`X-Forwarded-For`ヘッダーを設定できるため、IPスプーフィングの脆弱性の影響を受けやすくなります。

このミドルウェアはデフォルトでは有効になっていません。通常、このミドルウェアが必要かどうかは*わかる*はずで、正しいアーキテクチャで使用する際のセキュリティ面も認識しているはずです。確信が持てない場合は使用しないでください（`enable`を`false`のままにしてください）。


## Secure Headers

Locoには、`secure_headers`ミドルウェアによって適用されるデフォルトの安全なヘッダーが付属しています。これは、Railsエコシステムで[secure_headers](https://github.com/github/secure_headers)を使用して行われることと同様です。

`server.middleware`のYAMLセクションには、デフォルトで`github`プリセットがあります（これはGithubとTwitterが安全なヘッダーとして推奨するものです）。

```yaml
server:
  middleware:
    # set secure headers
    secure_headers:
      preset: github
```

特定のヘッダーをオーバーライドすることもできます：

```yaml
server:
  middleware:
    # set secure headers
    secure_headers:
      preset: github
      overrides:
        foo: bar
```

またはゼロから始めることもできます：

```yaml
server:
  middleware:
    # set secure headers
    secure_headers:
      preset: empty
      overrides:
        foo: bar
```

`htmx`をサポートするために、次のオーバーライドを追加して、一部のスクリプトのインライン実行を許可できます：

```yaml
secure_headers:
    preset: github
    overrides:
        # this allows you to use HTMX, and has unsafe-inline. Remove or consider in production
        "Content-Security-Policy": "default-src 'self' https:; font-src 'self' https: data:; img-src 'self' https: data:; object-src 'none'; script-src 'unsafe-inline' 'self' https:; style-src 'self' https: 'unsafe-inline'"
```

## Compression

`Loco`は[CompressionLayer](https://docs.rs/tower-http/0.5.0/tower_http/compression/index.html)を活用して、`ワンクリック`ソリューションを可能にします。

`accept-encoding`リクエストヘッダーに基づいてレスポンス圧縮を有効にするには、次のように設定を編集します：

```yaml
#...
  middlewares:
    compression:
      enable: true
```

これにより、各レスポンスが圧縮され、それに応じて`content-encoding`レスポンスヘッダーが設定されます。

## 事前圧縮されたアセット


`Loco`は[ServeDir::precompressed_gzip](https://docs.rs/tower-http/latest/tower_http/services/struct.ServeDir.html#method.precompressed_gzip)を活用して、事前圧縮されたアセットを提供する`ワンクリック`ソリューションを可能にします。

静的アセットがディスク上に`.gz`ファイルとして存在する場合、`Loco`はその場で圧縮する代わりにそれを提供します。

```yaml
#...
middlewares:
  ...
  static_assets:
    ...
    precompressed: true
```

## CORS
このミドルウェアは、HTTPリクエストで設定可能なオリジン、メソッド、ヘッダーを許可することで、Cross-Origin Resource Sharing（CORS）を有効にします。
ミドルウェア設定で定義されているように、寛容なCORSまたは特定のルールをサポートし、様々なアプリケーション要件に合わせて調整できます。

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

## ハンドラーとルートベースのミドルウェア

`Loco`では、特定のハンドラーやルートに[レイヤー](https://docs.rs/tower/latest/tower/trait.Layer.html)を適用することもできます。
ハンドラーとルートベースのミドルウェアの詳細については、[ミドルウェア](/docs/the-app/controller/#middleware)のドキュメントを参照してください。


### ハンドラーベースのミドルウェア：

`layer`メソッドを使用して特定のハンドラーにレイヤーを適用します。

```rust
// src/controllers/auth.rs
pub fn routes() -> Routes {
    Routes::new()
        .prefix("auth")
        .add("/register", post(register).layer(middlewares::log::LogLayer::new()))
}
```

### ルートベースのミドルウェア：

`layer`メソッドを使用して特定のルートにレイヤーを適用します。

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

# リクエスト検証
`JsonValidate`エクストラクターは、validatorクレートと統合することで入力[検証](https://github.com/Keats/validator)を簡素化します。受信リクエストデータを検証する方法の例を示します：

### 検証ルールの定義
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
### 検証付きハンドラーの作成
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
`JsonValidate`エクストラクターを使用すると、LocoはDataParams構造体に対して自動的に検証を実行します：
* 検証に合格した場合、ハンドラーはparamsを使用して実行を継続します。
* 検証に失敗した場合、400 Bad Requestレスポンスが返されます。

### 検証エラーのJSONとして返却
検証エラーを構造化されたJSON形式で返したい場合は、`JsonValidate`の代わりに`JsonValidateWithMessage`を使用します。レスポンス形式は次のようになります：

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

# ページネーション

多くのシナリオでは、データをクエリしてユーザーにレスポンスを返す際、ページネーションが重要です。`Loco`では、データをページネートし、APIレスポンスに一貫したページネーションレスポンススキーマを維持する簡単な方法を提供します。

`notes`エンティティおよび/またはスキャフォールドがあると仮定します（任意のエンティティに置き換えてください）。

## ページネーションの使用

```rust
use loco_rs::prelude::*;

let res = query::fetch_page(&ctx.db, notes::Entity::find(), &query::PaginationQuery::page(2)).await;
```


## フィルター付きページネーションの使用
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

- 取得したいエンティティを定義することから始めます。
- クエリ条件を作成します（この場合、タイトル列に「loco」を含む行をフィルタリング）。
- ページネーションパラメータを定義します。
- paginate関数を呼び出します。

### ページネーションビュー
前の例で`paginated_notes`を取得した後、モデルから返したいフィールドを選択し、すべての異なるデータレスポンスで同じページネーションレスポンスを維持できます。

Locoビューでユーザーに返すデータを定義します。ビューに慣れていない場合は、詳細なコンテキストについて[ドキュメント](@/docs/the-app/views.md)を参照してください。


`src/view/notes`にノートビューファイルを作成し、次のコードを記述します：

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
