+++
title = "Locoガイド"
date = 2021-05-01T08:00:00+00:00
updated = 2021-05-01T08:00:00+00:00
draft = false
weight = 3
sort_by = "weight"
template = "docs/page.html"

[extra]
toc = true
top = false
flair =[]
+++

## ガイドの前提条件

これは「長い道のり」のチュートリアルです。意図的に長く詳細にしており、手動でのビルド方法**と**ジェネレーターを使った自動ビルド方法の両方を学習できるように、構築スキルと仕組みの理解の両方を身につけられるようになっています。


### 名前の由来は？

`Loco`という名前は**loco**motiveから来ており、Railsへの敬意を表したもので、`loco`の方が`locomotive`より入力しやすいからです :-) また、いくつかの言語では「クレイジー」を意味しますが、それは本来の意図ではありません（あるいは、RustでRailsを構築するのはクレイジーなのでしょうか？時が教えてくれるでしょう！）。

### どの程度のRustの知識が必要ですか？

初心者レベルのRustに精通している必要がありますが、中級初心者レベル以上は必要ありません。Rustプロジェクトのビルド、テスト、実行方法を知っていて、`clap`、`regex`、`tokio`、`axum`などの人気ライブラリやその他のWebフレームワークを使ったことがある程度で十分です。特に高度なものは必要ありません。Locoには、動作原理を理解する必要があるような複雑なライフタイムやあまりにも魔法的なマクロはありません。


### Locoとは？

LocoはRailsに強くインスパイアされています。RailsとRustの両方を知っていれば、馴染みやすく感じるでしょう。Railsしか知らなくてRustが初めての場合、Locoを新鮮に感じるでしょう。Railsの知識は前提としていません。

<div class="infobox">
Railsはとても素晴らしいので、このガイドも<a href="https://guides.rubyonrails.org/getting_started.html">Railsガイド</a>に強くインスパイアされています
</div>

LocoはRust向けのWebまたはAPIフレームワークです。また、開発者向けの生産性スイートでもあります：趣味のプロジェクトや次のスタートアップを構築する際に必要なものがすべて含まれています。Railsに強くインスパイアされています。

- **MVCモデルのバリエーション**により、選択肢のパラドックスを排除します。アプリの構築に集中でき、どの抽象化を使うかといった学術的な決定を行う必要がありません。
- **ファットモデル、スリムコントローラー**。モデルにはロジックとビジネス実装の大部分を含み、コントローラーはHTTPを理解してパラメーターを移動させる軽量ルーターに留めるべきです。
- **コマンドライン駆動**により、勢いとフローを維持します。コピー&ペーストやゼロからのコーディングよりも、生成機能を活用します。
- **すべてのタスクが「インフラ対応」**、コードをプラグインして配線するだけです：コントローラー、モデル、ビュー、タスク、バックグラウンドジョブ、メーラーなど。
- **設定より規約**：決定はすでに行われています -- フォルダー構造、設定の形式と値、アプリの配線方法がアプリの動作と最も効果的な開発方法に影響します。

## 新しいLocoアプリの作成

このガイドに従ってステップバイステップの「ボトムアップ」学習を行うか、より早い「トップダウン」の紹介として[ツアー](@/docs/getting-started/tour/index.md)に進むこともできます。

### インストール

<!-- <snip id="quick-installation-command" inject_from="yaml" template="sh"> -->
```sh
cargo install loco
cargo install sea-orm-cli # DBが必要な場合のみ
```
<!-- </snip> -->


### 新しいLocoアプリの作成

これで新しいアプリを作成できます（組み込み認証のために「SaaS app」を選択してください）。

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



Locoがデフォルトで作成するもの一覧：

| ファイル/フォルダ | 目的                                                                                                                                                           |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/`         | コントローラー、モデル、ビュー、タスクなどが含まれています                                                                                                               |
| `app.rs`       | メインコンポーネント登録ポイント。重要な部分をここで配線します。                                                                                                  |
| `lib.rs`       | コンポーネントの様々なRust固有のエクスポート。                                                                                                                 |
| `bin/`         | `main.rs`ファイルがあります、これについて心配する必要はありません                                                                                                         |
| `controllers/` | コントローラーが含まれ、すべてのコントローラーは`mod.rs`経由でエクスポートされます                                                                                                   |
| `models/`      | モデルが含まれ、`models/_entities`には自動生成されたSeaORMモデル、`models/*.rs`にはモデル拡張ロジックが含まれ、`mod.rs`経由でエクスポートされます |
| `views/`       | JSONベースのビュー。API経由でJSONとして`serde`でシリアライズして出力できる構造体。                                                                          |
| `workers/`     | バックグラウンドワーカーがあります。                                                                                                                                      |
| `mailers/`     | メール送信用のメーラーロジックとテンプレート。                                                                                                                   |
| `fixtures/`    | データと自動フィクスチャ読み込みロジックが含まれています。                                                                                                                |
| `tasks/`       | メール送信、ビジネスレポート作成、DB メンテナンスなどの日常的なビジネス指向タスクが含まれています。                                         |
| `tests/`       | アプリ全体のテスト：モデル、リクエストなど。                                                                                                                       |
| `config/`      | ステージベースの設定フォルダ：development、test、production                                                                                                 |

## Hello, Loco!

いくつかのレスポンスを素早く取得してみましょう。そのためには、サーバーを起動する必要があります。

`myapp`に移動できます：

```sh
$ cd myapp
```

### サーバーの起動

<!-- <snip id="starting-the-server-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco start
```
<!-- </snip> -->

そして、動作していることを確認しましょう：

```sh
$ curl localhost:5150/_ping
{"ok":true}
```

組み込みの`_ping`ルートはロードバランサーにすべてが正常であることを知らせます。

必要なサービスがすべて稼働していることを確認しましょう：

```sh
$ curl localhost:5150/_health
{"ok":true}
```

<div class="infobox">
組み込みの<code>_health</code>ルートは、アプリが正しく設定されていることを確認します：データベースとRedisインスタンスへの接続を正常に確立できることを示します。
</div>

### "Hello", Loco

サービスに素早い_hello_レスポンスを追加してみましょう。

```sh
$ cargo loco generate controller guide --api
added: "src/controllers/guide.rs"
injected: "src/controllers/mod.rs"
injected: "src/app.rs"
added: "tests/requests/guide.rs"
injected: "tests/requests/mod.rs"
```

This is the generated controller body:

```rust
#![allow(clippy::missing_errors_doc)]
#![allow(clippy::unnecessary_struct_initialization)]
#![allow(clippy::unused_async)]
use loco_rs::prelude::*;
use axum::debug_handler;

#[debug_handler]
pub async fn index(State(_ctx): State<AppContext>) -> Result<Response> {
    format::empty()
}

pub fn routes() -> Routes {
    Routes::new()
        .prefix("api/guides/")
        .add("/", get(index))
}
```


Change the `index` handler body:

```rust
// replace
    format::empty()
// with this
    format::text("hello")
```

Start the server:

<!-- <snip id="starting-the-server-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco start
```
<!-- </snip> -->

Now, let's test it out:

```sh
$ curl localhost:5150/api/guides
hello
```

Loco has powerful generators, which will make you 10x productive and drive your momentum when building apps.

If you'd like to be entertained for a moment, let's "learn the hard way" and add a new controller manually as well.

Add a file called `home.rs`, and line `pub mod home;` in `mod.rs`:

```
src/
  controllers/
    auth.rs
    home.rs      <--- add this file
    users.rs
    mod.rs       <--- 'pub mod home;' the module here
```

Next, set up a _hello_ route, this is the contents of `home.rs`:

```rust
// src/controllers/home.rs
use loco_rs::prelude::*;

// _ctx contains your database connection, as well as other app resource that you'll need
async fn hello(State(_ctx): State<AppContext>) -> Result<Response> {
    format::text("ola, mundo")
}

pub fn routes() -> Routes {
    Routes::new().prefix("home").add("/hello", get(hello))
}
```

Finally, register this new controller routes in `app.rs`:

```rust
src/
  controllers/
  models/
  ..
  app.rs   <---- look here
```

Add the following in `routes()`:

```rust
// in src/app.rs
#[async_trait]
impl Hooks for App {
    ..
    fn routes() -> AppRoutes {
        AppRoutes::with_default_routes()
            .add_route(controllers::guide::routes())
            .add_route(controllers::auth::routes())
            .add_route(controllers::home::routes()) // <--- add this
    }
```

That's it. Kill the server and bring it up again:

<!-- <snip id="starting-the-server-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco start
```
<!-- </snip> -->

And hit `/home/hello`:

```sh
$ curl localhost:5150/home/hello
ola, mundo
```

You can take a look at all of your routes with:

```
$ cargo loco routes
  ..
  ..
[POST] /api/auth/login
[POST] /api/auth/register
[POST] /api/auth/reset
[POST] /api/auth/verify
[GET] /home/hello      <---- this is our new route!
  ..
  ..
$
```

<div class="infobox">
The <em>SaaS Starter</em> keeps routes under <code>/api</code> because it is client-side ready and we are using the <code>--api</code> option in scaffolding. <br/>
When using client-side routing like React Router, we want to separate backend routes from client routes: the browser will use <code>/home</code> but not <code>/api/home</code> which is the backend route, and you can call <code>/api/home</code> from the client with no worries. Nevertheless, the routes: <code>/_health</code> and <code>/_ping</code> are exceptions, they stay at the root.
</div>

## MVCとあなた

**従来のMVC（Model-View-Controller）はデスクトップUIプログラミングパラダイムから生まれました。** しかし、Webサービスへの適用性により急速に採用されました。MVCの黄金時代は2010年代初頭で、その後多くの他のパラダイムやアーキテクチャが登場しました。

**MVCはプロジェクトを簡素化するための非常に強力な原則とアーキテクチャであり**、Locoもこれに従っています。

WebサービスやAPIはHTMLやUIレスポンスを生成しないため_ビュー_の概念を持たないものの、**_安定した_、_安全な_サービスやAPIには確実にビューの概念がある**と私たちは主張します -- それはシリアライズされたデータ、その形状、互換性、バージョンです。

```
// a typical loco app contains all parts of MVC

src/
  controllers/
    users.rs
    mod.rs
  models/
    _entities/
      users.rs
      mod.rs
    users.rs
    mod.rs
  views/
    users.rs
    mod.rs
```

**This is an important _cognitive_ principle**. And the principle claims that you can only create safe, compatible API responses if you treat those as a separate, independently governed _thing_ -- hence the 'V' in MVC, in Loco.

<div class="infobox">
Models in Loco carry the same semantics as in Rails: <b>fat models, slim controllers</b>. This means that every time you want to build something -- <em>you reach out to a model</em>.
</div>

### モデルの生成

Locoのモデルはデータ*と*機能を表します。通常、データはデータベースに保存されます。アプリケーションのビジネスプロセスのほとんど、もしくはすべてがモデル（Active Recordとして）または複数のモデルのオーケストレーションとしてコーディングされます。

`Article`という新しいモデルを作成してみましょう：

```sh
$ cargo loco generate model article title:string content:text

added: "migration/src/m20231202_173012_articles.rs"
injected: "migration/src/lib.rs"
injected: "migration/src/lib.rs"
added: "tests/models/articles.rs"
injected: "tests/models/mod.rs"
```

### データベースマイグレーション

**スキーマの整合性を保つにはマイグレーションを使用します**。マイグレーションはデータベース構造への単一の変更です：完全なテーブル追加、変更、またはインデックス作成を含むことができます。

```rust
// this was generated into `migrations/` from the command:
//
// $ cargo loco generate model article title:string content:text
//
// it is automatically applied by Loco's migrator framework.
// you can also apply it manually using the command:
//
// $ cargo loco db migrate
//
#[async_trait::async_trait]
impl MigrationTrait for Migration {
    async fn up(&self, manager: &SchemaManager) -> Result<(), DbErr> {
        manager
            .create_table(
                table_auto_tz(Articles::Table)
                    .col(pk_auto(Articles::Id))
                    .col(string_null(Articles::Title))
                    .col(text(Articles::Content))
                    .to_owned(),
            )
            .await
    }

    async fn down(&self, manager: &SchemaManager) -> Result<(), DbErr> {
        manager
            .drop_table(Table::drop().table(Articles::Table).to_owned())
            .await
    }
}
```

You can recreate a complete database **by applying migrations in-series onto a fresh database** -- this is done automatically by Loco's migrator (which is derived from SeaORM).

When generating a new model, Loco will:

- Generate a new "up" database migration
- Apply the migration
- Reflect the entities from database structure and generate back your `_entities` code

You will find your new model as an entity, synchronized from your database structure in `models/_entities/`:

```
src/models/
├── _entities
│   ├── articles.rs  <-- sync'd from db schema, do not edit
│   ├── mod.rs
│   ├── prelude.rs
│   └── users.rs
├── articles.rs   <-- generated for you, your logic goes here.
├── mod.rs
└── users.rs
```

### Using `playground` to interact with the database

Your `examples/` folder contains:

- `playground.rs` - a place to try out and experiment with your models and app logic.

Let's fetch data using your models, using `playground.rs`:

```rust
// located in examples/playground.rs
// use this file to experiment with stuff
use loco_rs::{cli::playground, prelude::*};
// to refer to articles::ActiveModel, your imports should look like this:
use myapp::{app::App, models::_entities::articles};

#[tokio::main]
async fn main() -> loco_rs::Result<()> {
    let ctx = playground::<App>().await?;

    // add this:
    let res = articles::Entity::find().all(&ctx.db).await.unwrap();
    println!("{:?}", res);

    Ok(())
}

```

### Return a list of posts

In the example, we use the following to return a list:

```rust
let res = articles::Entity::find().all(&ctx.db).await.unwrap();
```

To see how to run more queries, go to the [SeaORM docs](https://www.sea-ql.org/SeaORM/docs/next/basic-crud/select/).

To execute your playground, run:

```rust
$ cargo playground
[]
```

Now, let's insert one item:

```rust
async fn main() -> loco_rs::Result<()> {
    let ctx = playground::<App>().await?;

    // add this:
    let active_model: articles::ActiveModel = articles::ActiveModel {
        title: Set(Some("how to build apps in 3 steps".to_string())),
        content: Set(Some("use Loco: https://loco.rs".to_string())),
        ..Default::default()
    };
    active_model.insert(&ctx.db).await.unwrap();

    let res = articles::Entity::find().all(&ctx.db).await.unwrap();
    println!("{:?}", res);

    Ok(())
}
```

And run the playground again:

```sh
$ cargo playground
[Model { created_at: ..., updated_at: ..., id: 1, title: Some("how to build apps in 3 steps"), content: Some("use Loco: https://loco.rs") }]
```

We're now ready to plug this into an `articles` controller. First, generate a new controller:

```sh
$ cargo loco generate controller articles --api
added: "src/controllers/articles.rs"
injected: "src/controllers/mod.rs"
injected: "src/app.rs"
added: "tests/requests/articles.rs"
injected: "tests/requests/mod.rs"
```

Edit `src/controllers/articles.rs`:

```rust
#![allow(clippy::unused_async)]
use loco_rs::prelude::*;

use crate::models::_entities::articles;

pub async fn list(State(ctx): State<AppContext>) -> Result<Response> {
    let res = articles::Entity::find().all(&ctx.db).await?;
    format::json(res)
}

pub fn routes() -> Routes {
    Routes::new().prefix("api/articles").add("/", get(list))
}
```

Now, start the app:

<!-- <snip id="starting-the-server-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco start
```
<!-- </snip> -->

And make a request:

```sh
$ curl localhost:5150/api/articles
[{"created_at":"...","updated_at":"...","id":1,"title":"how to build apps in 3 steps","content":"use Loco: https://loco.rs"}]
```

## CRUD APIの構築

次に、単一の記事の取得、削除、編集方法を見ていきます。IDによる記事の取得は`axum`の`Path`エクストラクターを使用して行います。

Replace the contents of `articles.rs` with this:

```rust
// this is src/controllers/articles.rs

#![allow(clippy::unused_async)]
use loco_rs::prelude::*;
use serde::{Deserialize, Serialize};

use crate::models::_entities::articles::{ActiveModel, Entity, Model};

#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct Params {
    pub title: Option<String>,
    pub content: Option<String>,
}

impl Params {
    fn update(&self, item: &mut ActiveModel) {
        item.title = Set(self.title.clone());
        item.content = Set(self.content.clone());
    }
}

async fn load_item(ctx: &AppContext, id: i32) -> Result<Model> {
    let item = Entity::find_by_id(id).one(&ctx.db).await?;
    item.ok_or_else(|| Error::NotFound)
}

pub async fn list(State(ctx): State<AppContext>) -> Result<Response> {
    format::json(Entity::find().all(&ctx.db).await?)
}

pub async fn add(State(ctx): State<AppContext>, Json(params): Json<Params>) -> Result<Response> {
    let mut item: ActiveModel = Default::default();
    params.update(&mut item);
    let item = item.insert(&ctx.db).await?;
    format::json(item)
}

pub async fn update(
    Path(id): Path<i32>,
    State(ctx): State<AppContext>,
    Json(params): Json<Params>,
) -> Result<Response> {
    let item = load_item(&ctx, id).await?;
    let mut item = item.into_active_model();
    params.update(&mut item);
    let item = item.update(&ctx.db).await?;
    format::json(item)
}

pub async fn remove(Path(id): Path<i32>, State(ctx): State<AppContext>) -> Result<Response> {
    load_item(&ctx, id).await?.delete(&ctx.db).await?;
    format::empty()
}

pub async fn get_one(Path(id): Path<i32>, State(ctx): State<AppContext>) -> Result<Response> {
    format::json(load_item(&ctx, id).await?)
}

pub fn routes() -> Routes {
    Routes::new()
        .prefix("api/articles")
        .add("/", get(list))
        .add("/", post(add))
        .add("/{id}", get(get_one))
        .add("/{id}", delete(remove))
        .add("/{id}", patch(update))
}
```

A few items to note:

- `Params` is a strongly typed required params data holder, and is similar in concept to Rails' _strongparams_, just safer.
- `Path(id): Path<i32>` extracts the `:id` component from a URL.
- Order of extractors is important and follows `axum`'s documentation (parameters, state, body).
- It's always better to create a `load_item` helper function and use it in all singular-item routes.
- While `use loco_rs::prelude::*` brings in anything you need to build a controller, you should note to import `crate::models::_entities::articles::{ActiveModel, Entity, Model}` as well as `Serialize, Deserialize` for params.


<div class="infobox">
The order of the extractors is important, as changing the order of them can lead to compilation errors. Adding the <code>#[debug_handler]</code> macro to handlers can help by printing out better error messages. More information about extractors can be found in the <a href="https://docs.rs/axum/latest/axum/extract/index.html#the-order-of-extractors">axum documentation</a>.
</div>


You can now test that it works, start the app:

<!-- <snip id="starting-the-server-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco start
```
<!-- </snip> -->

Add a new article:

```sh
$ curl -X POST -H "Content-Type: application/json" -d '{
  "title": "Your Title",
  "content": "Your Content xxx"
}' localhost:5150/api/articles
{"created_at":"...","updated_at":"...","id":2,"title":"Your Title","content":"Your Content xxx"}
```

Get a list:

```sh
$ curl localhost:5150/api/articles
[{"created_at":"...","updated_at":"...","id":1,"title":"how to build apps in 3 steps","content":"use Loco: https://loco.rs"},{"created_at":"...","updated_at":"...","id":2,"title":"Your Title","content":"Your Content xxx"}
```

### 2つ目のモデルの追加

別のモデルを追加しましょう。今度は`Comment`です。リレーションを作成したいと思います - コメントは投稿に属し、各投稿は複数のコメントを持つことができます。

Instead of coding the model and controller by hand, we're going to create a **comment scaffold** which will generate a fully working CRUD API comments. We're also going to use the special `references` type:

```sh
$ cargo loco generate scaffold comment content:text article:references --api
```

<div class="infobox">
The special <code>&lt;other_model&gt;:references:&lt;column_name&gt;</code> is also available. For when you want to have a different name for your column.
</div>

If you peek into the new migration, you'll discover a new database relation in the articles table:

```rust
      ..
      ..
  .col(integer(Comments::ArticleId))
  .foreign_key(
      ForeignKey::create()
          .name("fk-comments-articles")
          .from(Comments::Table, Comments::ArticleId)
          .to(Articles::Table, Articles::Id)
          .on_delete(ForeignKeyAction::Cascade)
          .on_update(ForeignKeyAction::Cascade),
  )
      ..
      ..
```


Now, lets modify our API in the following way:

1. Comments can be added through a shallow route: `POST comments/`
2. Comments can only be fetched in a nested route (forces a Post to exist): `GET posts/1/comments`
3. Comments cannot be updated, fetched singular, or deleted

In `src/controllers/comments.rs`, remove unneeded routes and functions:

```rust
pub fn routes() -> Routes {
    Routes::new()
        .prefix("api/comments")
        .add("/", post(add))
        // .add("/", get(list))
        // .add("/{id}", get(get_one))
        // .add("/{id}", delete(remove))
        // .add("/{id}", patch(update))
}
```

Also adjust the Params & update functions in `src/controllers/comments.rs`, by updating the scaffolded code marked with `<- add this`

```rust
pub struct Params {
    pub content: Option<String>,
    pub article_id: i32, // <- add this
}

impl Params {
    fn update(&self, item: &mut ActiveModel) {
        item.content = Set(self.content.clone());
        item.article_id = Set(self.article_id.clone()); // <- add this
    }
}
```

Now we need to fetch a relation in `src/controllers/articles.rs`. Add the following route:

```rust
pub fn routes() -> Routes {
  // ..
  // ..
  .add("/{id}/comments", get(comments))
}
```

And implement the relation fetching:

```rust
// to refer to comments::Entity, your imports should look like this:
use crate::models::_entities::{
    articles::{ActiveModel, Entity, Model},
    comments,
};

pub async fn comments(
    Path(id): Path<i32>,
    State(ctx): State<AppContext>,
) -> Result<Response> {
    let item = load_item(&ctx, id).await?;
    let comments = item.find_related(comments::Entity).all(&ctx.db).await?;
    format::json(comments)
}
```

<div class="infobox">
This is called "lazy loading", where we fetch the item first and later its associated relation. Don't worry - there is also a way to eagerly load comments along with an article.
</div>

Now start the app again:

<!-- <snip id="starting-the-server-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco start
```
<!-- </snip> -->

Add a comment to Article `1`:

```sh
$ curl -X POST -H "Content-Type: application/json" -d '{
  "content": "this rocks",
  "article_id": 1
}' localhost:5150/api/comments
{"created_at":"...","updated_at":"...","id":4,"content":"this rocks","article_id":1}
```

And, fetch the relation:

```sh
$ curl localhost:5150/api/articles/1/comments
[{"created_at":"...","updated_at":"...","id":4,"content":"this rocks","article_id":1}]
```

This ends our comprehensive _Guide to Loco_. If you made it this far, hurray!.

## タスク：データレポートのエクスポート

実世界のアプリは実世界の状況を処理する必要があります。例えば、ユーザーや顧客が何らかのレポートを必要とする場合があります。

You can:

- Connect to your production database, issue ad-hoc SQL queries. Or use some kind of DB tool. _This is unsafe, insecure, prone to errors, and cannot be automated_.
- Export your data to something like Redshift, or Google, and issue a query there. _This is a waste of resource, insecure, cannot be tested properly, and slow_.
- Build an admin. _This is time-consuming, and waste_.
- **Or build an adhoc task in Rust, which is quick to write, type safe, guarded by the compiler, fast, environment-aware, testable, and secure.**

This is where `cargo loco task` comes in.

First, run `cargo loco task` to see current tasks:

```sh
$ cargo loco task
seed_data		[Task for seeding data]
```

Generate a new task `user_report`

```sh
$ cargo loco generate task user_report

added: "src/tasks/user_report.rs"
injected: "src/tasks/mod.rs"
injected: "src/app.rs"
added: "tests/tasks/user_report.rs"
injected: "tests/tasks/mod.rs"
```

In `src/tasks/user_report.rs` you'll see the task that was generated for you. Replace it with following:

```rust
// find it in `src/tasks/user_report.rs`

use loco_rs::prelude::*;
use loco_rs::task::Vars;

use crate::models::users;

pub struct UserReport;

#[async_trait]
impl Task for UserReport {
    fn task(&self) -> TaskInfo {
      // description that appears on the CLI
        TaskInfo {
            name: "user_report".to_string(),
            detail: "output a user report".to_string(),
        }
    }

    // variables through the CLI:
    // `$ cargo loco task name:foobar count:2`
    // will appear as {"name":"foobar", "count":2} in `vars`
    async fn run(&self, app_context: &AppContext, vars: &Vars) -> Result<()> {
        let users = users::Entity::find().all(&app_context.db).await?;
        println!("args: {vars:?}");
        println!("!!! user_report: listing users !!!");
        println!("------------------------");
        for user in &users {
            println!("user: {}", user.email);
        }
        println!("done: {} users", users.len());
        Ok(())
    }
}
```

You can modify this task as you see fit. Access the models with `app_context`, or any other environmental resources, and fetch
variables that were given through the CLI with `vars`.

Running this task is done with:

```rust
$ cargo loco task user_report var1:val1 var2:val2 ...

args: Vars { cli: {"var1": "val1", "var2": "val2"} }
!!! user_report: listing users !!!
------------------------
done: 0 users
```
If you have not added a user before, the report will be empty.

To add a user check out chapter [Registering a New User](/docs/getting-started/tour/#registering-a-new-user) of [A Quick Tour with Loco](/docs/getting-started/tour/).

Remember: this is environmental, so you write the task once, and then execute in development or production as you wish. Tasks are compiled into the main app binary.

## 認証：リクエストの認証

`SaaS App`スターターを選択した場合、完全に設定された認証モジュールがアプリに組み込まれているはずです。
**コメントの追加**時に認証を要求する方法を見てみましょう。

Go back to `src/controllers/comments.rs` and take a look at the `add` function:

```rust
pub async fn add(State(ctx): State<AppContext>, Json(params): Json<Params>) -> Result<Response> {
    let mut item: ActiveModel = Default::default();
    params.update(&mut item);
    let item = item.insert(&ctx.db).await?;
    format::json(item)
}
```

To require authentication, we need to modify the function signature in this way:

```rust
async fn add(
    auth: auth::JWT,
    State(ctx): State<AppContext>,
    Json(params): Json<Params>,
) -> Result<Response> {
    // we only want to make sure it exists
    let _current_user = crate::models::users::Model::find_by_pid(&ctx.db, &auth.claims.pid).await?;

    // next, update
    // homework/bonus: make a comment _actually_ belong to user (user_id)
    let mut item: ActiveModel = Default::default();
    params.update(&mut item);
    let item = item.insert(&ctx.db).await?;
    format::json(item)
}
```
