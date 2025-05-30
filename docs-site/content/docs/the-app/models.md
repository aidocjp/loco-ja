+++
title = "Models"
description = ""
date = 2021-05-01T18:10:00+00:00
updated = 2024-01-07T21:10:00+00:00
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


`loco`のモデルは、データベースのクエリと書き込みを簡単に行うためのエンティティクラスを意味しますが、マイグレーションやシーディングも含みます。

## Sqlite vs Postgres 

新しいアプリを作成するときにデフォルトである`sqlite`を選択した可能性があります。Locoは`sqlite`と`postgres`間で_シームレス_に移行することができます。

開発には`sqlite`を使用し、本番には`postgres`を使用するのが一般的です。`pg`固有の機能を使用するため、開発と本番の両方で`postgres`を一貫して好む人もいます。最近では本番でも`sqlite`を使用する人もいます。いずれにしても、すべて有効な選択です。

`sqlite`の代わりに`postgres`を設定するには、`config/development.yaml`（または`production.yaml`）に移動し、アプリ名が`myapp`であると仮定して、以下を設定します：

```yaml
database:
  uri: "{{ get_env(name="DATABASE_URL", default="postgres://loco:loco@localhost:5432/myapp_development") }"
```

<div class="infobox">
ローカルのPostgresデータベースは <code>loco:loco</code> で、データベース名は <code>myapp_development</code> である必要があります。テストと本番環境では、データベース名はそれぞれ <code>myapp_test</code> と <code>myapp_production</code> である必要があります。
</div>

便宜上、PostgreSQLデータベースサーバーを起動するdockerコマンドを以下に示します：

<!-- <snip id="postgres-run-docker-command" inject_from="yaml" template="sh"> -->
```sh
docker run -d -p 5432:5432 \
  -e POSTGRES_USER=loco \
  -e POSTGRES_DB=myapp_development \
  -e POSTGRES_PASSWORD="loco" \
  postgres:15.3-alpine
```
<!-- </snip> -->



最後に、doctorコマンドを使用して接続を検証することもできます：

<!-- <snip id="doctor-command" inject_from="yaml template="sh"> -->
```sh
$ cargo loco doctor
    Finished dev [unoptimized + debuginfo] target(s) in 0.32s
    Running `target/debug/myapp-cli doctor`
✅ SeaORM CLI is installed
✅ DB connection: success
✅ Redis connection: success
```
<!-- </snip> -->

## Fat models, slim controllers

`loco`のモデルは**active recordに倣って設計されています**。これは、モデルがあなたの世界の中心点であり、アプリが持つすべてのロジックや操作がそこにあるべきことを意味します。

これは`User::create`がユーザーを作成することを意味しますが、**同時に**`user.buy(product)`が商品を購入することも意味します。

この方向性に同意する場合、以下のメリットを無料で得ることができます：

- **時間効率的なテスト**：モデルをテストすることで、ロジックと可動部分のほとんど、またはすべてをテストできるため。
- **_タスク_、ワーカー、その他の場所から**完全なアプリワークフローを実行する能力。
- モデルを組み合わせることで、機能とユースケースを効果的に**組み合わせる**ことができ、他には何も必要ありません。
- 本質的に、**モデルがあなたのアプリになり**、コントローラーはアプリを世界に公開する単なる一つの方法です。

ActiveRecord抽象化の背後にあるメインのORMとして[`SeaORM`](https://www.sea-ql.org/SeaORM/)を使用しています。

- _なぜDieselではないのか？_ - Dieselはより優れたパフォーマンスを持っていますが、そのマクロと一般的なアプローチは、私たちが実現しようとしていることと互換性がないと感じました
- _なぜsqlxではないのか_ - SeaORMは内部でsqlxを使用しているため、必要に応じて生の`sqlx`を使用するための配管はそこにあります。

## Example model

`loco`モデルのライフサイクルは_migration_から始まり、その後データベース構造から_entity_ Rustコードが自動的に生成されます：

```
src/
  models/
    _entities/   <--- autogenerated code
      users.rs   <--- the bare entity and helper traits
    users.rs  <--- your custom activerecord code
```

`users` activerecordの使用は、SeaORMで使用するのと同じです[例はこちらを参照](https://www.sea-ql.org/SeaORM/docs/next/basic-crud/select/)

`users` activerecordに機能を追加するには_拡張_を使用します：

```rust
impl super::_entities::users::ActiveModel {
    /// .
    ///
    /// # Errors
    ///
    /// .
    pub fn foobar(&self) -> Result<(), DbErr> {
        // implement and get back a `user.foobar()`
    }
}
```

# Crafting models

## The model generator

新しいモデルを追加するために、モデルジェネレーターはマイグレーションを作成し、それを実行し、その後データベーススキーマからエンティティ同期をトリガーして、モデルエンティティを作成し構築します。

```
$ cargo loco generate model posts title:string! content:text user:references
```

マイグレーション経由でモデルが追加されると、以下のデフォルトフィールドが提供されます：

- `created_at` (ts!): これはモデルが作成された時刻を示すタイムスタンプです。
- `updated_at` (ts!): これはモデルが更新された時刻を示すタイムスタンプです。

これらのフィールドは、マイグレーションコマンドで提供した場合は無視されます。

### Field syntax

各フィールドタイプには`!`または`^`のサフィックスを含めることができます：

- `!`はフィールドが**必須**であることを示します（つまり、データベースで`NOT NULL`）
- `^`はフィールドが**ユニーク**でなければならないことを示します。

サフィックスが使用されない場合、フィールドはnullにできます。


### Data types

スキーマデータタイプについて、スキーマを理解するために以下のマッピングを使用できます：

```rust
("uuid^", "uuid_uniq"),
("uuid", "uuid_null"),
("uuid!", "uuid"),
("string", "string_null"),
("string!", "string"),
("string^", "string_uniq"),
("text", "text_null"),
("text!", "text"),
("text^", "text_uniq"),
("small_unsigned^", "small_unsigned_uniq"),
("small_unsigned", "small_unsigned_null"),
("small_unsigned!", "small_unsigned"),
("big_unsigned^", "big_unsigned"),
("big_unsigned", "big_unsigned_null"),
("big_unsigned!", "big_unsigned_uniq"),
("small_int", "small_integer_null"),
("small_int!", "small_integer"),
("small_int^", "small_integer_uniq"),
("int", "integer_null"),
("int!", "integer"),
("int^", "integer_uniq"),
("big_int", "big_integer_null"),
("big_int!", "big_integer"),
("big_int^", "big_integer_uniq"),
("float", "float_null"),
("float!", "float"),
("float^", "float_uniq"),
("double", "double_null"),
("double!", "double"),
("double^", "double_uniq"),
("decimal", "decimal_null"),
("decimal!", "decimal"),
("decimal_len", "decimal_len_null"),
("decimal_len!", "decimal_len"),
("decimal^", "decimal_uniq"),
("bool", "boolean_null"),
("bool!", "boolean"),
("tstz", "timestamp_with_time_zone_null"),
("tstz!", "timestamp_with_time_zone"),
("date", "date_null"),
("date!", "date"),
("date^", "date_uniq"),
("date_time", "date_time_null"),
("date_time!", "date_time"),
("date_time^", "date_time_uniq"),
("blob", "blob_null"),
("blob!", "blob"),
("blob^", "blob_uniq"),
("json", "json_null"),
("json!", "json"),
("jsonb", "json_binary_null"),
("jsonb!", "json_binary"),
("jsonb^", "jsonb_uniq"),
("money", "money_null"),
("money!", "money"),
("money^", "money_uniq"),
("unsigned", "unsigned_null"),
("unsigned!", "unsigned"),
("unsigned^", "unsigned_uniq"),
("binary_len", "binary_len_null"),
("binary_len!", "binary_len"),
("binary_len^", "binary_len_uniq"),
("var_binary", "var_binary_null"),
("var_binary!", "var_binary"),
(" array", "array"),
(" array!", "array"),
(" array^", "array"),
```

Locoは、生成されるモデルと参照したいモデルの間の外部キー関係を定義するために`references`タイプを使用します。ただし、この特別なタイプを使用する方法は2つあることに注意してください：

1. `<other_model>:references`
2. `<other_model>:references:<column_name>`

最初のもの（`<other_model>:references`）は、セマンティクスから既に明らかなように、既存のモデル（この場合は`other_model`）への外部キー関係を作成するために使用されます。ただし、**フィールド名は暗黙的**です。

例えば、`post`という名前の新しいモデルを作成し、既に存在する`users`テーブル（マイグレーションが適用された新しいlocoプロジェクト内）を参照するフィールド/カラムを持たせたい場合、以下のコマンドを使用します：

```
cargo loco g model post title:string user:references
```

`user:references`を使用すると、特別な`<other_model>:references`タイプを使用し、`post`（新しいモデル）と`user`（既存のモデル）の間に関係を作成し、`posts`テーブルに`user_id`（暗黙的フィールド名）参照フィールドを追加します。

一方、2番目のアプローチ（`<other_model>:references:<column_name>`）を使用すると、好みに応じてフィールド/カラムに名前を付けることができる贅沢を得られます。したがって、前の例を取り上げて、タイトルとおそらく著者を指す外部キーを持つ`post`テーブルを作成したい場合、同じ前のコマンドを使用しますが、少し変更します：

```
cargo loco g model post title:string user:references:authored_by
```

`user:references:authored_by`を使用すると、特別な`<other_model>:references:<column_name>`タイプを使用し、`post`と`user`の間に関係を作成し、`user_id`の代わりに`authored_by`（明示的フィールド名）参照フィールドを`posts`テーブルに追加します。

空のモデルを生成できます：

```
$ cargo loco generate model posts
```


または、参照のないデータモデル：

```
$ cargo loco generate model posts title:string! content:text
```

## Migrations

モデルジェネレーターを使用する以外に、*マイグレーションを作成*することでスキーマを駆動します。

```
$ cargo loco generate migration <name of migration> [name:type, name:type ...]
```

これはプロジェクトのルートの`migration/`にマイグレーションを作成します。

適用できます：

```
$ cargo loco db migrate
```

そしてそこからエンティティ（Rustコード）を生成し直します：

```
$ cargo loco db entities
```

LocoはRailsと同様にマイグレーションファーストのフレームワークです。これは、モデル、データフィールド、またはモデル指向の変更を追加したいときに、それを説明するマイグレーションから始め、その後マイグレーションを適用して`model/_entities`で生成されたエンティティを取得することを意味します。

これは_everything-as-code_、_再現性_、_原子性_を強制し、スキーマの知識が失われることがありません。 

**マイグレーションの命名は重要**で、生成されるマイグレーションのタイプはマイグレーション名から推測されます。

### 新しいテーブルを作成

* 名前テンプレート: `Create___`
* 例: `CreatePosts`

```
$ cargo loco g migration CreatePosts title:string content:string
```

### カラムを追加

* 名前テンプレート: `Add___To___`
* 例: `AddNameAndAgeToUsers`（文字列`NameAndAge`は関係ありません、カラムは個別に指定しますが、`Users`はテーブル名になるため重要です）

```
$ cargo loco g migration AddNameAndAgeToUsers name:string age:int
```

### カラムを削除

* 名前テンプレート: `Remove___From___`
* 例: `RemoveNameAndAgeFromUsers`（_カラム追加_と同じ注意があります）

```
$ cargo logo g migration RemoveNameAndAgeFromUsers name:string age:int
```

### 参照を追加

* 名前テンプレート: `Add___RefTo___`
* 例: `AddUserRefToPosts`（`User`は関係ありません、参照は個別に1つまたは複数指定します。`Posts`はマイグレーションでテーブル名になるため重要です）

```
$ cargo loco g migration AddUserRefToPosts user:references
```

### 結合テーブルを作成

* 名前テンプレート: `CreateJoinTable___And___`（2つのテーブル間でサポート）
* 例: `CreateJoinTableUsersAndGroups`

```
$ cargo loco g migration CreateJoinTableUsersAndGroups count:int
```

関係に関するいくつかの状態カラム（ここでは`count`など）を追加することもできます。

### 空のマイグレーションを作成

上記のパターンのいずれにも当てはまらないマイグレーションには、説明的な名前を使用して空のマイグレーションを作成します。

```
$ cargo loco g migration FixUsersTable
```

### ダウンマイグレーション

間違いを犯したことに気づいた場合、いつでもマイグレーションを元に戻すことができます。これはマイグレーションによって行われた変更を元に戻します（マイグレーションに`down`の適切なコードを追加したと仮定します）。

<!-- <snip id="migrate-down-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco db down
```
<!-- </snip> -->

`down`コマンド単体では最後のマイグレーションのみをロールバックします。複数のマイグレーションをロールバックしたい場合は、ロールバックするマイグレーションの数を指定できます。

<!-- <snip id="migrate-down-n-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco db down 2
```
<!-- </snip> -->

### 動詞、単数形と複数形

- **references**: テーブル名には**単数形**を使用し、`<other_model>:references`タイプを使用します。`user:references`（`Users`を参照）、`vote:references`（`Votes`を参照）。`<other_model>:references:<column_name>`も利用可能です。`train:references:departing_train`（`Trains`を参照）。
- **カラム名**: 任意の名前。`snake_case`を推奨します。
- **テーブル名**: **複数形、スネークケース**。`users`、`draft_posts`。
- **マイグレーション名**: ファイル名として使用できるもの、スネークケースを推奨。`create_table_users`、`add_vote_id_to_movies`。
- **モデル名**: 自動的に生成されます。通常、生成される名前はパスカルケース、複数形です。`Users`、`UsersVotes`。

以下は命名規則を示す例です：

```sh
$ cargo loco generate model movies long_title:string user:references:added_by director:references
```

- モデル名は複数形: `movies`
- reference directorは単数形: `director:references`
- reference added_byは単数形の明示的な名前、参照されるモデルは単数形のまま: `user:references:added_by`
- カラム名はスネークケース: `long_title:string`

### マイグレーションの作成

マイグレーションDSLを使用するには、以下の`loco_rs::schema::*`インポートとSeaORMの`prelude`が必要です。

```rust
use loco_rs::schema::*;
use sea_orm_migration::prelude::*;
```

次に、構造体を作成します：

```rust
#[derive(DeriveMigrationName)]
pub struct Migration;
```

そして、マイグレーションを実装します（以下を参照）。

**テーブルの作成**

テーブルを作成する際は、2つの配列を提供します：(1) カラム (2) 参照。

参照フィールドを作成しない場合は、参照を空のままにします。

```rust
impl MigrationTrait for Migration {
    async fn up(&self, m: &SchemaManager) -> Result<(), DbErr> {
        create_table(
            m,
            "posts",
            &[
                ("title", ColType::StringNull),
                ("content", ColType::StringNull),
            ],
            &[],
        )
        .await
    }

    async fn down(&self, m: &SchemaManager) -> Result<(), DbErr> {
        drop_table(m, "posts").await
    }
}
```

**結合テーブルの作成**

参照を2番目の配列引数に提供します。空の文字列`""`を使用して、参照カラム名を自動生成したいことを示します（例：`user`参照は、`group_users`内の`user_id`カラムを通じて`users`テーブルに接続することを意味します）。

参照カラム名に特定の名前を指定したい場合は、空でない文字列を提供します。

```rust
impl MigrationTrait for Migration {
    async fn up(&self, m: &SchemaManager) -> Result<(), DbErr> {
        create_join_table(m, "group_users", &[], &[("user", ""), ("group", "")]).await
    }

    async fn down(&self, m: &SchemaManager) -> Result<(), DbErr> {
        drop_table(m, "group_users").await
    }
}
```

**カラムの追加**

単一のカラムを追加します。単一のマイグレーション内で、必要な数だけこのような文を使用できます（複数のカラムを追加するため）。


```rust
impl MigrationTrait for Migration {
    async fn up(&self, m: &SchemaManager) -> Result<(), DbErr> {
        add_column(m, "users", "amount", ColType::DecimalLenNull(24,8)).await?;
        Ok(())
    }

    async fn down(&self, m: &SchemaManager) -> Result<(), DbErr> {
        remove_column(m, "users", "amount").await?;
        Ok(())
    }
}
```


### 高度なマイグレーションの作成

`manager`を直接使用することで、マイグレーションを作成する際により高度な操作にアクセスできます。

**カラムの追加**

```rust
  manager
    .alter_table(
        Table::alter()
            .table(Movies::Table)
            .add_column_if_not_exists(integer(Movies::Rating))
            .to_owned(),
    )
    .await
```

**カラムの削除**

```rust
  manager
    .alter_table(
        Table::alter()
            .table(Movies::Table)
            .drop_column(Movies::Rating)
            .to_owned(),
    )
    .await
```

**インデックスの追加**

インデックスを追加するためのコードをコピーできます

```rust
  manager
    .create_index(
        Index::create()
            .name("idx-movies-rating")
            .table(Movies::Table)
            .col(Movies::Rating)
            .to_owned(),
    )
    .await;
```

**データ修正の作成**

マイグレーション内でデータ修正を作成するのは簡単です - 好きなようにSQL文を使用してください：

```rust
  async fn up(&self, manager: &SchemaManager) -> Result<(), DbErr> {

    let db = manager.get_connection();
    
    // issue SQL queries with `db`
    // https://www.sea-ql.org/SeaORM/docs/basic-crud/raw-sql/#use-raw-query--execute-interface

    Ok(())
  }
```

とはいえ、データ修正をコーディングする場所は以下から選択できます：

* `task` - 高レベルのモデルを使用できる場所
* `migration` - 構造を変更し、生のSQLでそれに起因するデータを修正できる場所
* またはアドホックな`playground` - 高レベルのモデルを使用したり、実験したりできる場所


## バリデーション

内部では[validator](https://docs.rs/validator)ライブラリを使用しています。まず、必要な制約でバリデーターを構築し、次に`ActiveModel`に`Validatable`を実装します。


<!-- <snip id="model-validation" inject_from="code" template="rust"> -->
```rust
#[derive(Debug, Validate, Deserialize)]
pub struct Validator {
    #[validate(length(min = 2, message = "Name must be at least 2 characters long."))]
    pub name: String,
    #[validate(email(message = "invalid email"))]
    pub email: String,
}

impl Validatable for super::_entities::users::ActiveModel {
    fn validator(&self) -> Box<dyn Validate> {
        Box::new(Validator {
            name: self.name.as_ref().to_owned(),
            email: self.email.as_ref().to_owned(),
        })
    }
}
```
<!-- </snip> -->


`Validatable`は、Locoにどの`Validator`を提供し、モデルからそれを構築する方法を指示する方法であることに注意してください。

これで、コード内で`user.validate()`をシームレスに使用できます。`Ok`の場合、モデルは有効であり、そうでない場合は`Err(...)`に検査可能なバリデーションエラーが見つかります。


## リレーションシップ

### 一対多

既存の`User`モデルに`Company`を関連付ける方法は次のとおりです。

```
$ cargo loco generate model company name:string user:references
```

これにより、`Company`に`User`を参照する`user_id`フィールドを持つマイグレーションが作成されます。


### 多対多

ここでは、多対多リンクテーブルで`User`と`Movie`をリンクする典型的な「投票」テーブルを作成する方法を示します。モデルジェネレーターで特別な`--link`フラグを使用することに注意してください。

新しい`Movie`エンティティを作成しましょう：

```
$ cargo loco generate model movies title:string
```

そして今度は、投票を記録するために`User`（既にある）と`Movie`（今生成した）の間のリンクテーブルを作成します：

```
$ cargo loco generate model --link users_votes user:references movie:references vote:int
    ..
    ..
Writing src/models/_entities/movies.rs
Writing src/models/_entities/users.rs
Writing src/models/_entities/mod.rs
Writing src/models/_entities/prelude.rs
... Done.
```

これにより、`user_id`と`movie_id`の両方を含む複合主キーを持つ`UsersVotes`という名前の多対多リンクテーブルが作成されます。正確に2つのIDを持つため、SeaORMはそれを多対多リンクテーブルとして識別し、適切な`via()`関係を持つエンティティを生成します：


```rust
// User, newly generated entity with a `via` relation at _entities/users.rs

// ..
impl Related<super::movies::Entity> for Entity {
    fn to() -> RelationDef {
        super::users_votes::Relation::Movies.def()
    }
    fn via() -> Option<RelationDef> {
        Some(super::users_votes::Relation::Users.def().rev())
    }
}
```

`via()`を使用すると、リンクテーブルの詳細を知る必要なく、`find_related`がリンクテーブルを通過します。




## 設定

利用可能なモデル設定は、開発、テスト、本番のすべての側面を制御し、本番経験から得られた多くの便利な機能を提供するため、非常にエキサイティングです。

<!-- <snip id="configuration-database" inject_from="code" template="yaml"> -->
```yaml
database:
  # Database connection URI
  uri: {{get_env(name="DATABASE_URL", default="postgres://loco:loco@localhost:5432/loco_app")}}
  # When enabled, the sql query will be logged.
  enable_logging: false
  # Set the timeout duration when acquiring a connection.
  connect_timeout: 500
  # Set the idle duration before closing a connection.
  idle_timeout: 500
  # Minimum number of connections for a pool.
  min_connections: 1
  # Maximum number of connections for a pool.
  max_connections: 1
  # Run migration up when application loaded
  auto_migrate: true
  # Truncate database when application loaded. This is a dangerous operation, make sure that you using this flag only on dev environments or test mode
  dangerously_truncate: false
  # Recreating schema when application loaded.  This is a dangerous operation, make sure that you using this flag only on dev environments or test mode
  dangerously_recreate: false
```
<!-- </snip>-->


これらのフラグを組み合わせることで、生産性を向上させるための異なる体験を作成できます。

アプリ起動前にトランケートできます -- これはテストの実行に役立ちます。または、アプリ起動時にDB全体を再作成できます -- これは統合テストや新しい環境のセットアップに役立ちます。本番環境では、これらをオフにしたいでしょう（そのため「dangerously」という部分があります）。

# シーディング

`Loco`には便利な`seeds`機能が搭載されており、素早く簡単なデータベースの再読み込みプロセスを効率化します。この機能は、開発環境やテスト環境での頻繁なリセット時に特に有用です。この機能の使い方を見てみましょう：

## 新しいシードの作成

### 1. 新しいシードファイルの作成

`src/fixtures`に移動し、新しいシードファイルを作成します。例えば：

```
src/
  fixtures/
    users.yaml
```

このyamlファイルには、挿入するデータベースレコードのセットを記載します。各レコードは、データベースの制約に基づいて必須のデータベースフィールドを含む必要があります。オプションの値は任意です。次のようなデータベースDDLがあるとします：

```sql
CREATE TABLE public.users (
	id serial4 NOT NULL,
	email varchar NOT NULL,
	"password" varchar NOT NULL,
	reset_token varchar NULL,
	created_at timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
	CONSTRAINT users_email_key UNIQUE (email),
	CONSTRAINT users_pkey PRIMARY KEY (id)
);
```

必須フィールドには`id`、`password`、`email`、`created_at`が含まれます。リセットトークンは空のままにできます。マイグレーションコンテンツファイルは次のようになります：

```yaml
---
- id: 1
  email: user1@example.com
  password: "$2b$12$gf4o2FShIahg/GY6YkK2wOcs8w4.lu444wP6BL3FyjX0GsxnEV6ZW"
  created_at: "2023-11-12T12:34:56.789"
- id: 2
  pid: 22222222-2222-2222-2222-222222222222
  email: user2@example.com
  reset_token: "SJndjh2389hNJKnJI90U32NKJ"
  password: "$2b$12$gf4o2FShIahg/GY6YkK2wOcs8w4.lu444wP6BL3FyjX0GsxnEV6ZW"
  created_at: "2023-11-12T12:34:56.789"
```

### シードの接続

以下の手順に従って、シードをアプリのHook実装に統合します：

1. アプリのHook実装に移動します。
2. seed関数の実装内にシードを追加します。Rustでの例は次のとおりです：

```rs
impl Hooks for App {
    // Other implementations...

    async fn seed(ctx: &AppContext, base: &Path) -> Result<()> {
        db::seed::<users::ActiveModel>(&ctx.db, &base.join("users.yaml").display().to_string()).await?;
        Ok(())
    }
}

```

この実装により、seed関数が呼び出されたときにシードが実行されることが保証されます。アプリケーションの構造と要件に基づいて詳細を調整してください。

## CLIを使用したシードの管理

- **データベースのリセット**  
  シードファイルをインポートする前にすべての既存データをクリアします。これは、古いデータが残らないようにして、新しいデータベース状態から始めたい場合に便利です。
- **データベーステーブルのファイルへのダンプ**  
  データベーステーブルの内容をファイルにエクスポートします。この機能により、データベースの現在の状態をバックアップしたり、環境間でデータを再利用するための準備ができます。

シードコマンドにアクセスするには、次のCLI構造を使用します：
<!-- <snip id="seed-help-command" inject_from="yaml" action="exec" template="sh"> -->
```sh
Seed your database with initial data or dump tables to files

Usage: demo_app-cli db seed [OPTIONS]

Options:
  -r, --reset                      Clears all data in the database before seeding
  -d, --dump                       Dumps all database tables to files
      --dump-tables <DUMP_TABLES>  Specifies specific tables to dump
      --from <FROM>                Specifies the folder containing seed files (defaults to 'src/fixtures') [default: src/fixtures]
  -e, --environment <ENVIRONMENT>  Specify the environment [default: development]
  -h, --help                       Print help
  -V, --version                    Print version
```
<!-- </snip> -->


### テストでの使用

1. テスト機能（`testing`）を有効にします

2. テストセクションで、以下の例に従います：

```rust
use loco_rs::testing::prelude::*;

#[tokio::test]
#[serial]
async fn handle_create_with_password_with_duplicate() {

    let boot = boot_test::<App, Migrator>().await;
    seed::<App>(&boot.app_context).await.unwrap();
    assert!(get_user_by_id(1).ok());
}
```

# マルチDB

`Loco`では、複数のデータベースを操作し、アプリケーション全体でインスタンスを共有できます。

## 追加DB

追加のデータベースをセットアップするには、データベース接続と設定から始めます。推奨されるアプローチは、設定ファイルに移動し、[settings](@/docs/the-app/your-project.md#settings)の下に次を追加することです：

```yaml
settings:
  extra_db:
    uri: postgres://loco:loco@localhost:5432/loco_app
    enable_logging: false
    connect_timeout: 500
    idle_timeout: 500
    min_connections: 1
    max_connections: 1
    auto_migrate: true
    dangerously_truncate: false
    dangerously_recreate: false
```



この[イニシャライザー](@/docs/extras/pluggability.md)を`initializers`フックにこの例のようにロードします

```rs
async fn initializers(ctx: &AppContext) -> Result<Vec<Box<dyn Initializer>>> {
        let  initializers: Vec<Box<dyn Initializer>> = vec![
            Box::new(loco_rs::initializers::extra_db::ExtraDbInitializer),
        ];

        Ok(initializers)
    }
```

これで、コントローラーでセカンダリデータベースを使用できます：

```rust
use sea_orm::DatabaseConnection;
use axum::{response::IntoResponse, Extension};

pub async fn list(
    State(ctx): State<AppContext>,
    Extension(secondary_db): Extension<DatabaseConnection>,
) -> Result<impl IntoResponse> {
  let res = Entity::find().all(&secondary_db).await;
}
```

## マルチDB（マルチテナント）

2つ以上の異なるデータベースに接続するには、データベース設定は次のようになります：
```yaml
settings:
  multi_db: 
    secondary_db:      
      uri: postgres://loco:loco@localhost:5432/loco_app
      enable_logging: false      
      connect_timeout: 500      
      idle_timeout: 500      
      min_connections: 1      
      max_connections: 1      
      auto_migrate: true      
      dangerously_truncate: false      
      dangerously_recreate: false
    third_db:      
      uri: postgres://loco:loco@localhost:5432/loco_app
      enable_logging: false      
      connect_timeout: 500      
      idle_timeout: 500      
      min_connections: 1      
      max_connections: 1      
      auto_migrate: true      
      dangerously_truncate: false      
      dangerously_recreate: false
```

次に、この[イニシャライザー](@/docs/extras/pluggability.md)を`initializers`フックにこの例のようにロードします

```rs
async fn initializers(ctx: &AppContext) -> Result<Vec<Box<dyn Initializer>>> {
        let  initializers: Vec<Box<dyn Initializer>> = vec![
            Box::new(loco_rs::initializers::multi_db::MultiDbInitializer),
        ];

        Ok(initializers)
    }
```

これで、コントローラーで複数のデータベースを使用できます：

```rust
use sea_orm::DatabaseConnection;
use axum::{response::IntoResponse, Extension};
use loco_rs::db::MultiDb;

pub async fn list(
    State(ctx): State<AppContext>,
    Extension(multi_db): Extension<MultiDb>,
) -> Result<impl IntoResponse> {
  let third_db = multi_db.get("third_db")?;
  let res = Entity::find().all(third_db).await;
}
```

# テスト

ジェネレーターを使用してモデルマイグレーションを作成した場合、`tests/models/posts.rs`に自動生成されたモデルテストもあるはずです（`post`という名前のモデルを生成したことを覚えていますか？）

典型的なテストには、テストデータのセットアップ、アプリの起動、テストコードが実行される前のデータベースの自動リセットに必要なすべてが含まれています。次のようになります：

```rust
use loco_rs::testing::prelude::*;

#[tokio::test]
#[serial]
async fn can_find_by_pid() {
    configure_insta!();

    let boot = boot_test::<App, Migrator>().await;
    seed::<App>(&boot.app_context).await.unwrap();

    let existing_user =
        Model::find_by_pid(&boot.app_context.db, "11111111-1111-1111-1111-111111111111").await;
    let non_existing_user_results =
        Model::find_by_email(&boot.app_context.db, "23232323-2323-2323-2323-232323232323").await;

    assert_debug_snapshot!(existing_user);
    assert_debug_snapshot!(non_existing_user_results);
}
```



テストプロセスを簡素化するために、`Loco`はテストの記述をより便利にする便利な関数を提供しています。`Cargo.toml`でテスト機能を有効にしてください：

```toml
[dev-dependencies]
loco-rs = { version = "*",  features = ["testing"] }
```


## データベースのクリーンアップ

場合によっては、クリーンなデータセットでテストを実行し、各テストが他のテストから独立し、以前のデータに影響されないようにしたいことがあります。この機能を有効にするには、`config/test.yaml`ファイルのdatabaseセクションで`dangerously_truncate`オプションをtrueに変更します。この設定により、Locoはboot appを実装する各テストの前にすべてのデータをトランケートします。

> ⚠️ 注意：特に本番環境では、意図しないデータ損失を避けるため、この機能を使用する際は注意してください。

- これを行う場合は、[serial](https://crates.io/crates/rstest)クレートを使用してすべての関連タスクを実行することをお勧めします。
- トランケートするテーブルを決定するには、エンティティモデルをAppフックに追加します：


```rust
pub struct App;
#[async_trait]
impl Hooks for App {
    //...
    async fn truncate(ctx: &AppContext) -> Result<()> {
        // truncate_table(&ctx.db, users::Entity).await?;
        Ok(())
    }

}
```

## 非同期
データベースデータを使用した非同期テストを記述する場合、あるテストが他のテストで使用されるデータに影響を与えないようにすることが重要です。非同期テストは同じデータベースデータセット上で同時に実行される可能性があるため、不安定なテスト結果につながる可能性があります。

同期テストのドキュメントで説明されている`boot_test`を使用する代わりに、`boot_test_with_create_db`関数を使用します。この関数はランダムなデータベーススキーマ名を生成し、テストが完了するとテーブルが削除されることを保証します。

注意：テスト実行を途中でキャンセルした場合（例：`Ctrl + C`を押した場合）、クリーンアッププロセスは実行されず、データベーステーブルは残ります。そのような場合は、手動で削除する必要があります。

```rust
use loco_rs::testing::prelude::*;

#[tokio::test]
async fn boot_test_with_create_db() {
    let boot = boot_test_with_create_db::<App, Migrator>().await;
}
```

## シーディング

```rust
use loco_rs::testing::prelude::*;

#[tokio::test]
#[serial]
async fn is_user_exists() {
    configure_insta!();

    let boot = boot_test::<App, Migrator>().await;
    seed::<App>(&boot.app_context).await.unwrap();
    assert!(get_user_by_id(1).ok());

}
```

このドキュメントは、Locoのテストヘルパーを活用するための詳細なガイドを提供し、データベースのクリーンアップ、スナップショットテストのためのデータクリーンアップ、テスト用データのシーディングをカバーしています。

## スナップショットテストデータのクリーンアップ

スナップショットテストでは、`created_date`、`id`、`pid`などの動的フィールドを持つデータ構造を比較することがよくあります。一貫したスナップショットを確保するために、Locoは正規表現置換を伴う定数データのリストを定義しています。これらの置換により、動的データをプレースホルダーに置き換えることができます。

スナップショット用の[insta](https://crates.io/crates/insta)を使用した例。

次の例では、すべてのユーザーモデルデータをクリーンアップする`cleanup_user_model`を使用できます。

```rust
use loco_rs::testing::prelude::*;

#[tokio::test]
#[serial]
async fn can_create_user() {
    request::<App, Migrator, _, _>(|request, _ctx| async move {
        // create user test
        with_settings!({
            filters => cleanup_user_model()
        }, {
            assert_debug_snapshot!(current_user_request.text());
        });
    })
    .await;
}

```

`CLEANUP_`で始まるクリーンアップ定数を直接使用することもできます。

## エンティティ生成のカスタマイズ

`Cargo.toml`の`[package.metadata.db.entity]`セクションに設定を追加することで、`sea-orm-cli`がエンティティを生成する方法をカスタマイズできます。例：

```toml
[package.metadata.db.entity]
max-connections = 1
ignore-tables = "table1,table2"
model-extra-derives = "CustomDerive"
```

この設定は、`cargo loco db entities`を実行する際に`sea-orm-cli generate entity`へのフラグとして渡されます。

`--output-dir`や`--database-url`のようないくつかのフラグは、Locoによって管理されているためオーバーライドできないことに注意してください。
