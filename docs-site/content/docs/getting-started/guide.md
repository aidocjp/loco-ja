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

生成されたコントローラーの本体は以下の通りです：

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


`index`ハンドラの本体を次のように変更します：

```rust
// 変更前
    format::empty()
// 変更後
    format::text("hello")
```

サーバーを起動します：

<!-- <snip id="starting-the-server-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco start
```
<!-- </snip> -->

では、動作を確認してみましょう：

```sh
$ curl localhost:5150/api/guides
hello
```

Locoには強力なジェネレーターがあり、アプリ開発の生産性と勢いを10倍に高めてくれます。

少し寄り道して「手動でやってみる」方法も学んでみましょう。新しいコントローラーを手作業で追加します。

`home.rs`というファイルを作成し、`mod.rs`に`pub mod home;`を追加します：

```
src/
  controllers/
    auth.rs
    home.rs      <--- add this file
    users.rs
    mod.rs       <--- 'pub mod home;' the module here
```

次に、_hello_ルートを設定します。`home.rs`の内容は以下の通りです：

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

最後に、この新しいコントローラーのルートを`app.rs`に登録します：

```rust
src/
  controllers/
  models/
  ..
  app.rs   <---- ここを編集
```

`routes()`に次を追加します：

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

これで完了です。サーバーを一度停止して再起動しましょう：

<!-- <snip id="starting-the-server-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco start
```
<!-- </snip> -->

そして `/home/hello` にアクセスします：

```sh
$ curl localhost:5150/home/hello
ola, mundo
```

すべてのルート一覧は次のコマンドで確認できます：

```
$ cargo loco routes
  ..
  ..
[POST] /api/auth/login
[POST] /api/auth/register
[POST] /api/auth/reset
[POST] /api/auth/verify
[GET] /home/hello      <---- これが新しいルートです！
  ..
  ..
$
```

<div class="infobox">
<em>SaaSスターター</em>は、クライアントサイド対応のため、ルートを<code>/api</code>配下にまとめています。これはスキャフォールディング時に<code>--api</code>オプションを使用しているためです。<br/>
React Routerのようなクライアントサイドルーティングを使う場合、バックエンドのルートとクライアントのルートを分離したいので、ブラウザは<code>/home</code>を利用しますが、<code>/api/home</code>（バックエンドルート）は利用しません。クライアントからは安心して<code>/api/home</code>を呼び出せます。ただし、<code>/_health</code>や<code>/_ping</code>のようなルートは例外で、ルート直下に配置されます。
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

**これは重要な「認知的」原則です。** この原則は、「APIレスポンスを独立したものとして扱うことで、安全で互換性のあるAPIレスポンスを作成できる」と主張しています。これがLocoにおけるMVCの「V（ビュー）」の意味です。

<div class="infobox">
LocoのモデルはRailsと同じ意味合いを持っています。<b>ファットモデル、スリムコントローラー</b>です。つまり、何かを作りたいときは<em>まずモデルにアクセスする</em>ということです。
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

新しいデータベースにマイグレーションを順番に適用することで、**完全なデータベースを再作成できます**。これはLocoのマイグレーター（SeaORM由来）が自動で行います。

新しいモデルを生成すると、Locoは次のことを行います：

- 新しい「up」マイグレーションを生成
- マイグレーションを適用
- データベース構造からエンティティを反映し、`_entities`コードを自動生成

新しいモデルは、データベース構造と同期されたエンティティとして`models/_entities/`に生成されます：

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

### playgroundを使ったデータベース操作

`examples/`フォルダには次のものが含まれています：

- `playground.rs` - モデルやアプリロジックを試すための実験用ファイルです。

`playground.rs`を使ってモデル経由でデータを取得してみましょう：

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

この例では、次のようにしてリストを返しています：

```rust
let res = articles::Entity::find().all(&ctx.db).await.unwrap();
```

他のクエリの実行方法は[SeaORMのドキュメント](https://www.sea-ql.org/SeaORM/docs/next/basic-crud/select/)を参照してください。

playgroundを実行するには次のコマンドを使います：

```rust
$ cargo playground
[]
```

次に、1件データを挿入してみましょう：

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

再度playgroundを実行してみましょう：

```sh
$ cargo playground
[Model { created_at: ..., updated_at: ..., id: 1, title: Some("how to build apps in 3 steps"), content: Some("use Loco: https://loco.rs") }]
```

このロジックを`articles`コントローラーに組み込む準備ができました。まずはコントローラーを生成します：

```sh
$ cargo loco generate controller articles --api
added: "src/controllers/articles.rs"
injected: "src/controllers/mod.rs"
injected: "src/app.rs"
added: "tests/requests/articles.rs"
injected: "tests/requests/mod.rs"
```

`src/controllers/articles.rs`を編集します：

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

アプリを起動しましょう：

<!-- <snip id="starting-the-server-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco start
```
<!-- </snip> -->

リクエストを送ってみます：

```sh
$ curl localhost:5150/api/articles
[{"created_at":"...","updated_at":"...","id":1,"title":"how to build apps in 3 steps","content":"use Loco: https://loco.rs"}]
```

## CRUD APIの構築

次に、単一の記事の取得、削除、編集方法を見ていきます。IDによる記事の取得は`axum`の`Path`エクストラクターを使用して行います。

`articles.rs`の内容を次のように置き換えます：

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

注意点をいくつか挙げます：

- `Params`は型安全な必須パラメータのデータホルダーで、Railsの _strongparams_ に似ていますが、より安全です。
- `Path(id): Path<i32>`はURLから`:id`部分を抽出します。
- エクストラクタの順序は重要で、`axum`のドキュメント（パラメータ、state、bodyの順）に従います。
- 単一アイテム用のルートでは`load_item`ヘルパー関数を作って使うのがベストです。
- `use loco_rs::prelude::*`でコントローラーに必要なものはほぼ揃いますが、`crate::models::_entities::articles::{ActiveModel, Entity, Model}`やパラメータ用の`Serialize, Deserialize`もインポートしてください。

<div class="infobox">
エクストラクタの順序は重要です。順序を変えるとコンパイルエラーになることがあります。ハンドラに<code>#[debug_handler]</code>マクロを付けると、より分かりやすいエラーメッセージが表示されます。エクストラクタの詳細は<a href="https://docs.rs/axum/latest/axum/extract/index.html#the-order-of-extractors">axumのドキュメント</a>を参照してください。
</div>


動作確認のため、アプリを起動します：

<!-- <snip id="starting-the-server-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco start
```
<!-- </snip> -->

新しい記事を追加してみましょう：

```sh
$ curl -X POST -H "Content-Type: application/json" -d '{
  "title": "Your Title",
  "content": "Your Content xxx"
}' localhost:5150/api/articles
{"created_at":"...","updated_at":"...","id":2,"title":"Your Title","content":"Your Content xxx"}
```

記事一覧を取得します：

```sh
$ curl localhost:5150/api/articles
[{"created_at":"...","updated_at":"...","id":1,"title":"how to build apps in 3 steps","content":"use Loco: https://loco.rs"},{"created_at":"...","updated_at":"...","id":2,"title":"Your Title","content":"Your Content xxx"}
```

### 2つ目のモデルの追加

別のモデルを追加しましょう。今度は`Comment`です。リレーションを作成したいと思います - コメントは投稿に属し、各投稿は複数のコメントを持つことができます。

モデルやコントローラーを手作業で書く代わりに、**commentスキャフォールド**を使って完全なCRUD APIコメントを自動生成します。ここでは特別な`references`型も使います：

```sh
$ cargo loco generate scaffold comment content:text article:references --api
```

<div class="infobox">
特別な<code>&lt;other_model&gt;:references:&lt;column_name&gt;</code>も利用できます。カラム名を別の名前にしたい場合に使えます。
</div>

新しく生成されたマイグレーションを覗くと、articlesテーブルに新しいデータベースリレーションが追加されていることが分かります：

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


次に、APIを以下のように修正します：

1. コメントはshallowルート（`POST comments/`）で追加できる
2. コメントの取得はネストしたルート（`GET posts/1/comments`）のみ（必ずPostが存在する）
3. コメントの更新・単体取得・削除は不可

`src/controllers/comments.rs`で不要なルートや関数を削除します：

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

また、`src/controllers/comments.rs`のParamsやupdate関数も、スキャフォールドされたコードの`<- add this`部分を修正します：

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

次に、`src/controllers/articles.rs`でリレーションを取得するルートを追加します：

```rust
pub fn routes() -> Routes {
  // ..
  // ..
  .add("/{id}/comments", get(comments))
}
```

そしてリレーション取得の実装を追加します：

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
これは「遅延読み込み（lazy loading）」と呼ばれる方法で、まずアイテムを取得し、その後に関連するリレーションを取得します。なお、記事と一緒にコメントを一括で取得する「イーガーロード（eager loading）」の方法もありますのでご安心ください。
</div>

アプリを再度起動しましょう：

<!-- <snip id="starting-the-server-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco start
```
<!-- </snip> -->

Article `1` にコメントを追加します：

```sh
$ curl -X POST -H "Content-Type: application/json" -d '{
  "content": "this rocks",
  "article_id": 1
}' localhost:5150/api/comments
{"created_at":"...","updated_at":"...","id":4,"content":"this rocks","article_id":1}
```

そしてリレーションを取得します：

```sh
$ curl localhost:5150/api/articles/1/comments
[{"created_at":"...","updated_at":"...","id":4,"content":"this rocks","article_id":1}]
```

これでLocoの包括的なガイドは終了です。ここまでたどり着いたあなた、おめでとうございます！

## タスク：データレポートのエクスポート

実世界のアプリは実世界の状況を処理する必要があります。例えば、ユーザーや顧客が何らかのレポートを必要とする場合があります。

例えば、次のような方法が考えられます：

- 本番データベースに接続して、その場しのぎのSQLクエリを発行する。もしくはDBツールを使う。_これは危険でセキュリティ上問題があり、ミスも起きやすく自動化もできません。_
- データをRedshiftやGoogleなどにエクスポートして、そこでクエリを発行する。_これはリソースの無駄で、セキュリティも不十分、テストも困難で遅いです。_
- 管理画面を作る。_これは時間がかかり、無駄が多いです。_
- **あるいはRustでアドホックなタスクを書きましょう。これは素早く書けて型安全、コンパイラで守られ、高速で環境対応・テスト可能・安全です。**

ここで`cargo loco task`の出番です。

まず、`cargo loco task`を実行して現在のタスク一覧を確認します：

```sh
$ cargo loco task
seed_data		[Task for seeding data]
```

新しいタスク`user_report`を生成します：

```sh
$ cargo loco generate task user_report

added: "src/tasks/user_report.rs"
injected: "src/tasks/mod.rs"
injected: "src/app.rs"
added: "tests/tasks/user_report.rs"
injected: "tests/tasks/mod.rs"
```

`src/tasks/user_report.rs`には自動生成されたタスクが記載されています。次の内容に置き換えてください：

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

このタスクは自由にカスタマイズできます。`app_context`を使ってモデルや他の環境リソースにアクセスしたり、CLIで渡された変数は`vars`から取得できます。

このタスクの実行は次の通りです：

```rust
$ cargo loco task user_report var1:val1 var2:val2 ...

args: Vars { cli: {"var1": "val1", "var2": "val2"} }
!!! user_report: listing users !!!
------------------------
done: 0 users
```
まだユーザーを追加していない場合、レポートは空になります。

ユーザーの追加方法は [A Quick Tour with Loco](/docs/getting-started/tour/#registering-a-new-user) の「Registering a New User」章を参照してください。

この仕組みは環境依存なので、一度タスクを書けば開発環境でも本番環境でも同じように実行できます。タスクはメインアプリのバイナリにコンパイルされます。

## 認証：リクエストの認証

`SaaS App`スターターを選択した場合、完全に設定された認証モジュールがアプリに組み込まれているはずです。
**コメントの追加**時に認証を要求する方法を見てみましょう。

`src/controllers/comments.rs`に戻り、`add`関数を確認します：

```rust
pub async fn add(State(ctx): State<AppContext>, Json(params): Json<Params>) -> Result<Response> {
    let mut item: ActiveModel = Default::default();
    params.update(&mut item);
    let item = item.insert(&ctx.db).await?;
    format::json(item)
}
```

認証を必須にするには、関数シグネチャを次のように修正します：

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
