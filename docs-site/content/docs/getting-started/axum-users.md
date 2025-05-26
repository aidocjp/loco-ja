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

### Locoへの移行

Locoでは、`config/`フォルダでプールの値を設定するだけです。デフォルトで最適な値が自動的に設定されるため、特に指定しなくても問題ありませんが、必要に応じて以下のように設定できます：

```yaml
database:
  enable_logging: false
  connect_timeout: 500
  idle_timeout: 500
  min_connections: 1
  max_connections: 1
```

### 判定

* **書くコードなし** - DBプールの値選択や設定ミスのリスクから解放されます
* **変更が簡単** - 本番環境で負荷に応じて値を変えたい場合も、Axumのみだと再コンパイル・再デプロイが必要ですが、Locoなら設定を変えてプロセスを再起動するだけです。


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


### Locoへの移行

1:1でそのままコピーペーストしたい場合は、Axumのルートをそのままコピーし、Locoの`after_routes()`フックにルーターを差し込むだけです：

```rust
  async fn after_routes(router: AxumRouter, _ctx: &AppContext) -> Result<AxumRouter> {
      // AxumRouterを使ってルートをマウントし、AxumRouterを返す
  }

```

Locoにルートのメタデータ情報を認識させたい場合（後で役立つことがあります）、各コントローラーで`routes()`関数を次のように記述します：

```rust
// Axumのみを使う場合の一般的な例
pub fn router() -> Router {
  Router::new()
        .route("/auth/register", post(create_user))
        .route("/auth/login", post(login_user))
}

// Locoでの記述例（`Routes`と`add`を使う点に注目）
pub fn routes() -> Routes {
  Routes::new()
      .add("/auth/register", post(create_user))
      .add("/auth/login", post(login_user))
}
```

### 判定

* **ドロップイン互換** - LocoはAxumをベースにしており、すべての構成要素をそのまま活かせるため、既存のAxumコードをそのまま利用できます。
* **ルートメタデータも無料で取得** - Axumルーターの弱点である「現在設定されているルートの一覧化や自動OpenAPIスキーマ生成」も、Locoの小さなメタデータレイヤーでサポート。`Routes`を使えば自動的にメタデータが付与され、Axumルーターとの互換性も維持されます。
