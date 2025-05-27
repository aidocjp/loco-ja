+++
title = "Views"
description = ""
date = 2021-05-01T18:10:00+00:00
updated = 2021-05-01T18:10:00+00:00
draft = false
weight = 4
sort_by = "weight"
template = "docs/page.html"

[extra]
lead = ""
toc = true
top = false
flair =[]
+++

`Loco`では、Webリクエストの処理はコントローラー、モデル、ビューに分割されています。

- **コントローラー**はリクエストの解析とペイロードを処理し、その後モデルに制御が流れます
- **モデル**は主にデータベースとの通信と必要に応じてCRUD操作を実行します。また、すべてのビジネスロジックとドメインロジック、操作をモデル化します。
- **ビュー**はクライアントに送り返される最終レスポンスを組み立て、レンダリングする責任を負います。

JSONレスポンスである_JSONビュー_、またはテンプレートビューエンジンによって動力を得て最終的にHTMLレスポンスになる_テンプレートビュー_を選択できます。両方を組み合わせることもできます。

<div class="infobox">
This is similar in spirit to Rails' `jbuilder` views which are JSON, and regular views, which are HTML, only that in LOCO we focus on being JSON-first.
</div>

## JSON views

例として、ユーザーログインを処理するエンドポイントがあります。ユーザーが有効な場合、`user`モデルを`LoginResponse`ビュー（JSONビュー）に渡してレスポンスを返すことができます。

3つのステップがあります：

1. リクエストを解析、受け入れる
2. ドメインオブジェクトを作成：モデル
3. ドメインモデルを最終レスポンスを**形作る**ビューオブジェクトに渡す

以下のRustコードは、ユーザーログインリクエストを処理する責任を持つコントローラーを表し、レスポンスの_形作り_を`LoginResponse`に任せます。

```rust
use crate::{views::auth::LoginResponse};
async fn login(
    State(ctx): State<AppContext>,
    Json(params): Json<LoginParams>,
) -> Result<Response> {
    // Fetching the user model with the requested parameters
    // let user = users::Model::find_by_email(&ctx.db, &params.email).await?;

    // Formatting the JSON response using LoginResponse view
    format::json(LoginResponse::new(&user, &token))
}
```

一方、`LoginResponse`は`serde`によって動力を得るレスポンスシェイピングビューです：

```rust
use serde::{Deserialize, Serialize};

use crate::models::_entities::users;

#[derive(Debug, Deserialize, Serialize)]
pub struct LoginResponse {
    pub token: String,
    pub pid: String,
    pub name: String,
}

impl LoginResponse {
    #[must_use]
    pub fn new(user: &users::Model, token: &String) -> Self {
        Self {
            token: token.to_string(),
            pid: user.pid.to_string(),
            name: user.name.clone(),
        }
    }
}

```

## Template views

ユーザーにHTMLを返したい場合は、サーバーサイドテンプレートを使用します。これはRubyの`erb`やNodeの`ejs`、あるいはPHPの動作と似ています。

サーバーサイドテンプレートレンダリングのために、人気の[Tera](http://keats.github.io/tera/)テンプレートエンジンに基づいたビルトインの`TeraView`エンジンを提供しています。

<div class="infobox">
To use this engine you need to verify that you have a <code>ViewEngineInitializer</code> in <code>initializers/view_engine.rs</code> which is also specified in your <code>app.rs</code>. If you used the SaaS Starter, this should already be configured for you.
</div>

Teraビューエンジンは新しい`assets/`フォルダーからリソースを取得します。以下は構造の例です：

```
assets/
├── i18n
│   ├── de-DE
│   │   └── main.ftl
│   ├── en-US
│   │   └── main.ftl
│   └── shared.ftl
├── static
│   ├── 404.html
│   └── image.png
└── views
    └── home
        └── hello.html
config/
:
src/
├── controllers/
├── models/
:
└── views/
```

### Creating a new view

まず、テンプレートを作成します。この場合、`assets/views/home/hello.html`にTeraテンプレートを追加します。**assets/**はプロジェクトのルート（`src/`と`config/`の隣）にあることに注意してください。

```html
<html>
  <body>
    find this tera template at <code>assets/views/home/hello.html</code>:
    <br />
    <br />
    {{ /* t(key="hello-world", lang="en-US") */ }},
    <br />
    {{ /* t(key="hello-world", lang="de-DE") */ }}
  </body>
</html>
```

次に、`src/views/dashboard.rs`でこのテンプレートをカプセル化する強型付けされた`view`を作成します：

```rust
// src/views/dashboard.rs
use loco_rs::prelude::*;

pub fn home(v: impl ViewRenderer) -> Result<impl IntoResponse> {
    format::render().view(&v, "home/hello.html", data!({}))
}

```

そして、`src/views/mod.rs`に追加します：

```rust
pub mod dashboard;
```

次に、コントローラーに移動してビューを使用します：

```rust
// src/controllers/dashboard.rs
use loco_rs::prelude::*;

use crate::views;

pub async fn render_home(ViewEngine(v): ViewEngine<TeraView>) -> Result<impl IntoResponse> {
    views::dashboard::home(v)
}

pub fn routes() -> Routes {
    Routes::new().prefix("home").add("/", get(render_home))
}

```

最後に、`src/app.rs`で新しいコントローラーのルートを登録します

```rust
pub struct App;
#[async_trait]
impl Hooks for App {
    // omitted for brevity

    fn routes(_ctx: &AppContext) -> AppRoutes {
        AppRoutes::with_default_routes()
            .add_route(controllers::auth::routes())
            // include your controller's routes here
            .add_route(controllers::dashboard::routes())
    }
```

上記のすべてを完了したら、`cargo loco routes`を実行したときに新しいルートを確認できるはずです

```
$ cargo loco routes
[GET] /_health
[GET] /_ping
[POST] /api/auth/forgot
[POST] /api/auth/login
[POST] /api/auth/register
[POST] /api/auth/reset
[POST] /api/auth/verify
[GET] /api/auth/current
[GET] /home              <-- the corresponding URL for our new view
```

### どのように動作するのか？

- `ViewEngine`は`loco_rs::prelude::*`経由で利用可能なエクストラクターです
- `TeraView`はLocoで提供されるTeraビューエンジンで、これも`loco_rs::prelude::*`経由で利用可能です
- コントローラーはリクエストを受け取り、いくつかのモデルロジックを呼び出し、その後ビューに**モデルやその他のデータ**を提供することを扱い、ビューがどのように動作するかは気にしません
- `views::dashboard::home`は不透明な呼び出しで、ビューがどのように動作するか、またはバイトがどのようにブラウザーに達するかの詳細を隠しています。これは_良いこと_です
- ビューエンジンを交換したい場合、ここのカプセル化は魔法のように機能します。エクストラクタータイプを変更できます：`ViewEngine<Foobar>`、そしてすべてが機能します。なぜなら`v`は最終的にただの`ViewRenderer`トレイトだからです

### Static assets

静的アセットを提供し、ビューテンプレートでそれらを参照したい場合は、_Static Middleware_を使用し、次のように設定します：

```yaml
static:
  enable: true
  must_exist: true
  precompressed: false
  folder:
    uri: "/static"
    path: "assets/static"
  fallback: "assets/static/404.html"
```

テンプレートでは、静的リソースを次のように参照できます：

```html
<img src="/static/image.png" />
```

ただし、静的ミドルウェアが機能するためには、デフォルトのフォールバックが無効になっていることを確認してください：

```yaml
fallback:
  enable: false
```

### Customizing the Tera view engine

Teraビューエンジンは以下の設定で提供されます：

- テンプレートのロードと場所： `assets/**/*.html`
- Teraビューエンジンに設定された国際化（i18n）、テンプレートで使用する翻訳関数を利用できます： `t(..)`

`i18n`ライブラリの設定詳細を変更したい場合は、`src/initializers/view_engine.rs`を編集できます。

初期化子を編集することで、以下ができます：

- カスタムTera関数を追加
- `i18n`ライブラリを削除
- Teraまたは`i18n`ライブラリの設定を変更
- 新しいまたはカスタムのTera（おそらく異なるバージョン）インスタンスを提供

### Using your own view engine

ビューエンジンとしてTeraが気に入らない、またはHandlebarsやその他を使用したい場合は、独自のカスタムビューエンジンを非常に簡単に作成できます。

以下はダミーの「Hello」ビューエンジンの例です。これは常に_hello_という単語を返すビューエンジンです。

```rust
// src/initializers/hello_view_engine.rs
use axum::{Extension, Router as AxumRouter};
use async_trait::async_trait;
use loco_rs::{
    app::{AppContext, Initializer},
    controller::views::{ViewEngine, ViewRenderer},
    Result,
};
use serde::Serialize;

#[derive(Clone)]
pub struct HelloView;
impl ViewRenderer for HelloView {
    fn render<S: Serialize>(&self, _key: &str, _data: S) -> Result<String> {
        Ok("hello".to_string())
    }
}

pub struct HelloViewEngineInitializer;
#[async_trait]
impl Initializer for HelloViewEngineInitializer {
    fn name(&self) -> String {
        "custom-view-engine".to_string()
    }

    async fn after_routes(&self, router: AxumRouter, _ctx: &AppContext) -> Result<AxumRouter> {
        Ok(router.layer(Extension(ViewEngine::from(HelloView))))
    }
}
```

これを使用するには、`src/app.rs`フックに追加する必要があります：

```rust
// src/app.rs
// add your custom "hello" view engine in the `initializers(..)` hook
impl Hooks for App {
    // ...
    async fn initializers(_ctx: &AppContext) -> Result<Vec<Box<dyn Initializer>>> {
        Ok(vec![
            // ,.----- add it here
            Box::new(initializers::hello_view_engine::HelloViewEngineInitializer),
        ])
    }
    // ...
```

### Tera Built-ins

Locoは[built-ins](https://keats.github.io/tera/docs/#built-ins)関数とともにTeraを含んでいます。さらに、Locoは以下のカスタムビルトイン関数を導入しています：

Locoビルトイン関数を見るには：

- [numbers](https://docs.rs/loco-rs/latest/loco_rs/controller/views/tera_builtins/filters/number/index.html)

## Embedded Assets Feature

LocoのEmbedded Assets機能を使用すると、すべての静的アセットをアプリケーションバイナリに直接バンドルできます。つまり、CSS、画像、PDFなどを含む`assets`フォルダー以下のすべてが、単一の実行ファイルの一部になります。

この機能を使用するには、`Cargo.toml`で`loco-rs`をインポートするときに`embedded_assets`機能を有効にする必要があります：

```toml
[dependencies]
loco-rs = { version = "...", features = ["embedded_assets"] }
```

### メリット

- **単一バイナリデプロイ**: 単一ファイルを配布するだけでよいため、デプロイを簡素化します。よりシンプルなデプロイでは、別々のアセットディレクトリやCDN設定を心配する必要がありません。
- **原子更新**: アプリケーションを更新するとき、アセットはコードと原子的に更新され、コードとアセット間の不一致の可能性を減らします。
- **潜在的に高速なロード時間**: アセットはメモリから直接ロードされ、特にディスクI/Oが遅い環境ではファイルシステムから読み取るよりも高速になる可能性があります。

### 考慮事項

- **バイナリサイズの増加**: アセットを埋め込むと、アプリケーションバイナリのサイズが自然に大きくなります。
- **アセット変更による再コンパイル**: アセットへのいかなる変更もアプリケーションの再コンパイルを必要とします。アセットが頻繁に変更される場合、これは開発ワークフローを遅くする可能性があります。

### Seamlessly Switching Modes

コントローラーやビューでのコード変更なしに、埋め込みアセットの使用とファイルシステムからのアセット提供を簡単に切り替えることができます。切り替えは`embedded_assets`機能フラグの有無によって処理されます。

ただし、埋め込みアセットを使用_しない_場合（つまり、ファイルシステムから提供する場合）にTeraが正しく機能することを保証するために、以前にカスタマイズした場合、`src/initializers/view_engine.rs`ファイルに必要なTera関数登録のみが含まれていることを確認する必要があります。特に、翻訳関数`t`については、`loco_rs::tera_helpers::FluentLoader`を使用していない場合、初期化子が次のようになっていることを確認してください：

```rust
tera_engine
    .tera
    .register_function("t", FluentLoader::new(arc));
```

あるいは、アプリケーション内で内部機能フラグを導入して、アセットのロード方法やTeraの設定方法を切り替え、より細かい制御を提供することもできます。

### Build Time Logs

`embedded_assets`機能を有効にしてアプリケーションをビルドすると、Locoは`assets`ディレクトリをスキャンし、発見されたファイルを埋め込みます。ビルドプロセス中に以下のようなログが表示され、どのアセットが含まれているかを示します：

```
warning: loco-rs@0.15.0: Assets will only be loaded from the application directory
warning: loco-rs@0.15.0: Discovered directories for assets:
warning: loco-rs@0.15.0:   - /path/to/your/myapp/assets
warning: loco-rs@0.15.0:   - /path/to/your/myapp/assets/static
warning: loco-rs@0.15.0:   - /path/to/your/myapp/assets/i18n
warning: loco-rs@0.15.0:   - /path/to/your/myapp/assets/i18n/de-DE
warning: loco-rs@0.15.0:   - /path/to/your/myapp/assets/i18n/en-US
warning: loco-rs@0.15.0:   - /path/to/your/myapp/assets/views
warning: loco-rs@0.15.0:   - /path/to/your/myapp/assets/views/home
warning: loco-rs@0.15.0: Found asset: /path/to/your/myapp/assets/static/styles.css -> /static/styles.css
warning: loco-rs@0.15.0: Found asset: /path/to/your/myapp/assets/static/dummy.pdf -> /static/dummy.pdf
warning: loco-rs@0.15.0: Found asset: /path/to/your/myapp/assets/static/404.html -> /static/404.html
warning: loco-rs@0.15.0: Found asset: /path/to/your/myapp/assets/views/base.html -> base.html
warning: loco-rs@0.15.0: Found 13 asset files
warning: loco-rs@0.15.0: Generated code for 6 static assets and 7 templates
```

この出力は、Locoがアセットファイル（CSS、PDF、HTMLテンプレートなど）を発見し、それらをバイナリに埋め込むための必要なコードを生成したことを確認します。パスはプロジェクトの構造を反映します。
