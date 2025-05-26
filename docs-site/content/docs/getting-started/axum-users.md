+++
title = "Axum vs Loco"
description = "AxumからLocoへの移行方法を示します"
date = 2023-12-01T19:30:00+00:00
updated = 2023-12-01T19:30:00+00:00
draft = false
weight = 5
sort_by = "weight"
template = "docs/page.html"

[extra]
toc = true
top = false
flair =[]
+++

<div class="infobox">
<b>注意：LocoはAxumベースで、「バッテリー同梱のAxum」です</b>。AxumコードをLocoに移行するのは非常に簡単です。
</div>

API、実際のデータベース、実際のシナリオ、および設定やロギングなどの実際の運用要件を使用した実世界のプロジェクトを記述しようとするAxumベースのアプリである[realworld-axum-sqlx](https://github.com/launchbadge/realworld-axum-sqlx)を研究します。

`realworld-axum-sqlx`を一つずつ分解することで、**AxumからLocoに移行することで、コードの大部分がすでにあなたのために書かれていることを示します**。より良いベストプラクティス、より良い開発体験、統合テスト、コード生成、そしてより高速なアプリ構築を得ることができます。

**この分解を使用して**、独自のAxumベースアプリをLocoに移行する方法も理解できます。質問がある場合は、[ディスカッション](https://github.com/loco-rs/loco/discussions)に連絡するか、[緑の招待ボタンをクリックしてdiscord](https://github.com/loco-rs/loco)に参加してください。

## `main`

Axumを使用する際、アプリのすべてのコンポーネントをセットアップし、ルーターを取得し、ミドルウェアを追加し、コンテキストを設定し、最終的にソケットで`listen`をセットアップする独自の`main`関数が必要です。

これは多くの手動で、エラーが起こりやすい作業です。

Locoでは以下ができます：

* 設定で必要なミドルウェアのオン/オフを切り替え
* `cargo loco start`を使用、`main`ファイルは一切不要
* 本番環境では、実行する`your_app`という名前のコンパイル済みバイナリが得られます


### Locoへの移行

* Loco `config/`で必要なミドルウェアをセットアップ

```yaml
server:
  middlewares:
    limit_payload:
      body_limit: 5mb
  # .. 以下にさらにミドルウェア ..
```

* Loco `config/`でサーバーポートを設定

```yaml
server:
  port: 5150
```

### 判定

* **書くコードなし**、必要でない限りmain関数を手動でコーディングする必要がありません
* **すぐに使えるベストプラクティス**、すべてのLocoアプリで共有される統一されたmainファイルのベストプラクティスが得られます
* **変更が簡単**、テストのためにミドルウェアを削除/追加したい場合、設定でスイッチを切り替えるだけで、リビルドは不要です


## 環境変数

realworld axumコードベースは[dotenv](https://github.com/launchbadge/realworld-axum-sqlx/blob/main/.env.sample)を使用しており、`main`で明示的な読み込みが必要です：

```rust
 dotenv::dotenv().ok();
```

また、`.env`ファイルが利用可能で、保守され、読み込まれる必要があります：

```
DATABASE_URL=postgresql://postgres:{password}@localhost/realworld_axum_sqlx
HMAC_KEY={random-string}
RUST_LOG=realworld_axum_sqlx=debug,tower_http=debug
```

これはプロジェクトで提供される**サンプル**ファイルで、手動でコピーして編集する必要があり、多くの場合非常にエラーが起こりやすいものです。

### Locoへの移行

Loco：標準の`config/[stage].yaml`設定を使用し、`get_env`を使って環境から特定の値を読み込みます


```yaml
# config/development.yaml

# Webサーバー設定
server:
  # サーバーがリッスンするポート。サーバーバインディングは0.0.0.0:{PORT}
  port:  {{/* get_env(name="NODE_PORT", default=5150) */}}
```

この設定は強く型付けされており、データベースURL、ロガーレベルとフィルタリングなど、最もよく使われる値が含まれています。推測したり車輪の再発明をする必要はありません。

### 判定

* **コーディング不要**、Locoに移行する際に書くコードが減ります
* **可動部分が少ない**、Axumのみを使用する場合、環境変数に加えて設定を用意する必要がありますが、これはLocoで無料で得られるものです

## データベース

Axumのみを使用する場合、通常、接続、プール、ルートで使用できるようにセットアップする必要があります。以下は、通常`main.rs`に記述するコードです：

```rust
    let db = PgPoolOptions::new()
        .max_connections(50)
        .connect(&config.database_url)
        .await
        .context("could not connect to database_url")?;
```

Then you have to hand-wire this connection
```rust
 .layer(AddExtensionLayer::new(ApiContext {
                config: Arc::new(config),
                db,
            }))
```

### Moving to Loco

In Loco you just set your values for the pool in your `config/` folder. We already pick up best effort default values so you don't have to do it, but if you want to, this is how it looks like:


```yaml
database:
  enable_logging: false
  connect_timeout: 500
  idle_timeout: 500
  min_connections: 1
  max_connections: 1
```

### Verdict

* **No code to write** - save yourself the dangers of picking the right values for your db pool, or misconfiguring it
* **Change is easy** - often you want to try different values under different loads in production, with Axum only, you have to recompile, redeploy. With Loco you can set a config and restart the process.


## ロギング

アプリ全体で、ロギングストーリーを手動でコーディングする必要があります。どれを選びますか？`tracing`か`slog`か？ロギングかトレーシングか？どちらが良いのでしょうか？

Here's what exists in the real-world-axum project. In serving:

```rust
  // Enables logging. Use `RUST_LOG=tower_http=debug`
  .layer(TraceLayer::new_for_http()),
```

And in `main`:

```rust
    // Initialize the logger.
    env_logger::init();
```

And ad-hoc logging in various points:

```rust
  log::error!("SQLx error: {:?}", e);
```

### Moving to Loco

In Loco, we've already answered these hard questions and provide multi-tier logging and tracing:

* Inside the framework, internally
* Configured in the router
* Low level DB logging and tracing
* All of Loco's components such as tasks, background jobs, etc. all use the same facility

And we picked `tracing` so that any and every Rust library can "stream" into your log uniformly. 

But we also made sure to create smart filters so you don't get bombarded with libraries you don't know, by default.

You can configure your logger in `config/`

```yaml
logger:
  enable: true
  pretty_backtrace: true
  level: debug
  format: compact
```

### Verdict

* **No code to write** - no set up code, no decision to make. We made the best decision for you so you can write more code for your app.
* **Build faster** - you get traces for only what you want. You get error backtraces which are colorful, contextual, and with zero noise which makes it easier to debug stuff. You can change formats and levels for production.
* **Change is easy** - often you want to try different values under different loads in production, with Axum only, you have to recompile, redeploy. With Loco you can set a config and restart the process.

## ルーティング

AxumからLocoへのルート移行は実際にドロップインです。LocoはネイティブのAxumルーターを使用します。

ルートリストや情報などの機能が必要な場合、Axumルーターに変換されるネイティブのLocoルーターを使用するか、独自のAxumルーターを使用できます。


### Moving to Loco

If you want 1:1 complete copy-paste experience, just copy your Axum routes, and plug your router in Loco's `after_routes()` hook:

```rust
  async fn after_routes(router: AxumRouter, _ctx: &AppContext) -> Result<AxumRouter> {
      // use AxumRouter to mount your routes and return an AxumRouter
  }

```

If you want Loco to understand the metadata information about your routes (which can come in handy later), write your `routes()` function in each of your controllers in this way:


```rust
// this is what people usually do using Axum only
pub fn router() -> Router {
  Router::new()
        .route("/auth/register", post(create_user))
        .route("/auth/login", post(login_user))
}

// this is how it looks like using Loco (notice we use `Routes` and `add`)
pub fn routes() -> Routes {
  Routes::new()
      .add("/auth/register", post(create_user))
      .add("/auth/login", post(login_user))
}
```

### Verdict

* **A drop-in compatibility** - Loco uses Axum and keeps all of its building blocks intact so that you can just use your own existing Axum code with no efforts.
* **Route metadata for free** - one gap that Axum routers has is the ability to describe the currently configured routes, which can be used for listing or automatic OpenAPI schema generation. Loco has a small metadata layer to support this. If you use `Routes` you get it for free, while all of the different signatures remain compatible with Axum router.
