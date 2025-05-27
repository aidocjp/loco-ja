+++
title = "Your Project"
description = ""
date = 2021-05-01T18:10:00+00:00
updated = 2024-01-07T21:10:00+00:00
draft = false
weight = 2
sort_by = "weight"
template = "docs/page.html"

[extra]
lead = ""
toc = true
top = false
flair =[]
+++

## `cargo loco`で開発を進める

スターターアプリを作成します：

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

アプリに`cd`して、さまざまなコマンドを試してみましょう：

<!-- <snip id="help-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco --help
```
<!-- </snip> -->

<!-- <snip id="exec-help-command" inject_from="yaml" action="exec" template="sh"> -->
```sh
The one-person framework for Rust

Usage: demo_app-cli [OPTIONS] <COMMAND>

Commands:
  start       Start an app
  db          Perform DB operations
  routes      Describe all application endpoints
  middleware  Describe all application middlewares
  task        Run a custom task
  jobs        Managing jobs queue
  scheduler   Run the scheduler
  generate    code generation creates a set of files and code templates based on a predefined set of rules
  doctor      Validate and diagnose configurations
  version     Display the app version
  watch       Watch and restart the app
  help        Print this message or the help of the given subcommand(s)

Options:
  -e, --environment <ENVIRONMENT>  Specify the environment [default: development]
  -h, --help                       Print help
  -V, --version                    Print version
```
<!-- </snip> -->


CLIを通じて開発を進めることができるようになりました：

```
$ cargo loco generate model posts
$ cargo loco generate controller posts
$ cargo loco db migrate
$ cargo loco start
```

テストの実行やRustでの作業は、あなたが既に知っている通りです：

```
$ cargo build
$ cargo test
```

### アプリの起動

アプリを実行するには、以下を実行します：

<!-- <snip id="starting-the-server-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco start
```
<!-- </snip> -->

### Background workers

設定（`config/`内）に基づいて、ワーカーはどのように動作するかを知っています：

```yaml
workers:
  # requires Redis
  mode: BackgroundQueue

  # can also use:
  # ForegroundBlocking - great for testing
  # BackgroundAsync - for same-process jobs, using tokio async
```

そして、実際のプロセスをさまざまな方法で実行できます：

- `rr start --worker` - ワーカーのみを実行し、バックグラウンドジョブを処理します。これはスケールに適しています。`rr start`でサービスアプリを実行し、その後`rr start --worker`で多数のプロセスベースのワーカーを任意のマシンに分散して実行できます。

* `rr start --server-and-worker` - 同じunixプロセスでサービスとバックグラウンドワーカープロセッサーの両方を実行します。バックグラウンドジョブの実行にTokioを使用します。これは、大きなコストをかけずに単一サーバーで実行したい場合や、リソースが制約されている場合に適しています。

### Getting your app version

アプリがコンパイルされ、その後本番にコピーされるため、Locoは2つの重要な運用情報を提供します：

* このアプリのバージョン、およびどのGIT SHAからビルドされたか？ `cargo loco version`
* このアプリはどのLocoバージョンに対してコンパイルされたか？ `cargo loco --version`

両方のバージョン文字列は解析可能で安定しているため、統合スクリプト、監視ツールなどで使用できます。

`src/app.rs`ファイルの`app_version`フックをオーバーライドすることで、独自のカスタムアプリバージョニングスキームを形作ることができます。


## Using the scaffold generator

スキャフォールディングは、アプリケーションの主要コンポーネントを生成する効率的で迅速な方法です。スキャフォールディングを利用することで、新しいリソースのモデル、ビュー、コントローラーを一度に作成できます。


scaffoldコマンドを確認してください：
<!-- <snip id="scaffold-help-command" inject_from="yaml" action="exec" template="sh"> -->
```sh
Generates a CRUD scaffold, model and controller

Usage: demo_app-cli generate scaffold [OPTIONS] <NAME> [FIELDS]...

Arguments:
  <NAME>       Name of the thing to generate
  [FIELDS]...  Model fields, eg. title:string hits:int

Options:
  -k, --kind <KIND>                The kind of scaffold to generate [possible values: api, html, htmx]
      --htmx                       Use HTMX scaffold
      --html                       Use HTML scaffold
      --api                        Use API scaffold
  -e, --environment <ENVIRONMENT>  Specify the environment [default: development]
  -h, --help                       Print help
  -V, --version                    Print version
```
<!-- </snip> -->

単一のブログ投稿を表すPostリソースのスキャフォールドを生成することから始めることができます。これを達成するには、ターミナルを開いて以下のコマンドを入力します：
<!-- <snip id="scaffold-post-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco generate scaffold posts name:string title:string content:text --api
```
<!-- </snip> -->

scaffold generateコマンドは、scaffoldコマンドに`--template`フラグを追加することでAPI、HTML、HTMXをサポートします。

### スキャフォールドファイルレイアウト

スキャフォールドジェネレーターはアプリケーションにいくつかのファイルを作成します：

| ファイル    | 目的                                                                                                                                    |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| `migration/src/lib.rs`                     |  Postマイグレーションを含む。                                                                                |
| `migration/src/m20240606_102031_posts.rs`  | Postsマイグレーション。                                                                                        |
| `src/app.rs`                               | アプリケーションルーターにPostsを追加。                                                                     |
| `src/controllers/mod.rs`                   | Postsコントローラーを含む。                                                                           |
| `src/controllers/posts.rs`                 | Postsコントローラー。                                                                                   |
| `tests/requests/posts.rs`                  | 機能テスト。                                                                                     |
| `src/models/mod.rs`                        | Postsモデルを含む。                                                                                  |
| `src/models/posts.rs`                      | Postsモデル。                                                                                            |
| `src/models/_entities/mod.rs`              | Posts Sea-ormエンティティモデルを含む。                                                                    |
| `src/models/_entities/posts.rs`            | Sea-ormエンティティモデル。                                                                                   |
| `src/views/mod.rs`                         | Postsビューを含む。HTMLおHTMXテンプレートのみ。                                                |
| `src/views/posts.rs`                       | Postsテンプレートジェネレーター。HTMLおHTMXテンプレートのみ。                                             |
| `assets/views/posts/create.html`           | Post作成テンプレート。HTMLおHTMXテンプレートのみ。                                                 |
| `assets/views/posts/edit.html`             | Post編集テンプレート。HTMLおHTMXテンプレートのみ。                                                   |
| `assets/views/posts/list.html`             | Post一覧テンプレート。HTMLおHTMXテンプレートのみ。                                                   |
| `assets/views/posts/show.html`             | Post表示テンプレート。HTMLおHTMXテンプレートのみ。                                                   |

## アプリの設定
デフォルトでは、locoはconfig/ディレクトリに設定ファイルを保存します。、3つの環境に対して事前定義された設定を提供します：

```
config/
  development.yaml
  production.yaml
  test.yaml
```

環境は以下に基づいて自動的に選択されます：

- コマンドラインフラグ：`cargo loco start --environment production`、指定されない場合はフォールバック
- `LOCO_ENV`または`RAILS_ENV`または`NODE_ENV`

何も指定されない場合、デフォルト値は`development`です。

`Loco`フレームワークは、デフォルト環境に加えてカスタム環境をサポートします。カスタム環境を追加するには、前の例で使用された環境識別子と一致する名前の設定ファイルを作成します。

### デフォルト設定パスのオーバーライド
カスタム設定ディレクトリを使用するには、`LOCO_CONFIG_FOLDER`環境変数を希望のフォルダーパスに設定します。これにより、`loco`はデフォルトの`config/`フォルダーの代わりに指定されたディレクトリから設定ファイルをロードするように指示されます。

### 設定内のプレースホルダー/変数

設定ファイルに値を注入することができます。この例では、`NODE_PORT`環境変数からポート値を取得しています：

```yaml
# config/development.yaml
# every configuration file is a valid Tera template
server:
  # Port on which the server will listen. the server binding is 0.0.0.0:{PORT}
  port:  {{/* get_env(name="NODE_PORT", default=5150) */}}
  # The UI hostname or IP address that mailers will point to.
  host: http://localhost
  # Out of the box middleware configuration. to disable middleware you can changed the `enable` field to `false` of comment the middleware block
```

[get_env](https://keats.github.io/tera/docs/#get-env)関数はTeraテンプレートエンジンの一部です。他に何が使用できるかを確認するには[Tera](https://keats.github.io/tera/docs)ドキュメントを参照してください。

### 例

'qa'環境を追加したいと仮定します。configフォルダーに`qa.yaml`ファイルを作成します：

```
config/
  development.yaml
  production.yaml
  test.yaml
  qa.yaml
```

'qa'環境を使用してアプリケーションを実行するには、以下のコマンドを実行します：
<!-- <snip id="starting-the-server-command-with-environment-env-var" inject_from="yaml" template="sh"> -->
```sh
LOCO_ENV=qa cargo loco start
```
<!-- </snip> -->

### Settings

設定ファイルはLocoアプリを設定するためのノブを含んでいます。`settings:`セクションでカスタム設定を持つこともできます。`config/development.yaml`に`settings:`セクションを追加します
<!-- <snip id="configuration-settings" inject_from="code" template="yaml"> -->
```yaml
settings:
  allow_list:
    - google.com
    - apple.com
```
<!-- </snip> -->

これらの設定は`ctx.config.settings`に`serde_json::Value`として表示されます。構造体を追加することで、強型付けされた設定を作成できます：

```rust
// put this in src/common/settings.rs
#[derive(Serialize, Deserialize, Default, Debug)]
pub struct Settings {
    pub allow_list: Option<Vec<String>>,
}

impl Settings {
    pub fn from_json(value: &serde_json::Value) -> Result<Self> {
        Ok(serde_json::from_value(value.clone())?)
    }
}
```

そして、以下のようにどこからでも設定にアクセスできます：


```rust
// in controllers, workers, tasks, or elsewhere,
// as long as you have access to AppContext (here: `ctx`)

if let Some(settings) = &ctx.config.settings {
    let settings = common::settings::Settings::from_json(settings)?;
    println!("allow list: {:?}", settings.allow_list);
}
```

### Server



`server:`下のインターフェース（リスニングなど）パラメーターの詳細な説明を以下に示します：

* `port:` 名前の通りポートを変更するため、主にロードバランサーの背後にある場合など。

* `binding:` IPインターフェースが「バインド」する先を変更するため。主に`nginx`のようなロードバランサーの背後にいる場合はローカルアドレスにバインドします（LBもそこにある場合）。ただし、「世界」（`0.0.0.0`）にバインドすることもできます。binding:フィールドは設定またはCLI（`-b`フラグを使用）経由で設定できます -- これはRailsが行っていることです。

* `host:` - 「可視性」ユースケースやアウトオブバンドユースケースのため。例えば、現在のサーバーホスト（ドメイン名などの観点から）を表示したい場合があり、これは可視性のために役立ちます。そして、メールの場合のように -- サーバーアドレスは「アウトオブバンド」であり、Gmailアカウントを開いてあなたのメールがあるとき -- あなたの外部アドレスや可視アドレス（公式ドメイン名など）のように見えるものをクリックする必要があり、間違ったことをする可能性がある内部「ホスト」アドレスではありません（「http://127.0.0.1/account/verify」を指すメールリンクを想像してみてください）



### Logger

YAMLファイルの`logger:`セクションのコメント化されたフィールド以外に、さらなるコンテキストを以下に示します：

* `logger.pretty_backtrace` - 優れた開発体験のためにノイズのないカラフルなバックトレースを表示します。これはプロセスのenvに`RUST_BACKTRACE=1`を強制的に設定し、特定のエラーで（コストのかかる）バックトレースキャプチャを有効にすることに注意してください。開発ではこれを有効にし、本番では無効にしてください。本番で必要な場合は、コマンドラインで`RUST_BACKTRACE=1`をアドホックに使用して表示してください。


利用可能なすべての設定オプションについては[こちらをクリック](https://docs.rs/loco-rs/latest/loco_rs/config/struct.Config.html)してください
