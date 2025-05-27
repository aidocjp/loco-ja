+++
title = "Upgrades"
description = ""
date = 2021-05-01T18:20:00+00:00
updated = 2021-05-01T18:20:00+00:00
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

## 新しいLocoバージョンがリリースされたときは？

- コードリポジトリでクリーンなブランチを作成します。
- メインの`Cargo.toml`でLocoのバージョンを更新します
- [CHANGELOG](https://github.com/loco-rs/loco/blob/master/CHANGELOG.md)を確認し、必要な破壊的変更とリファクタリングを見つけます（もしあれば）。
- プロジェクト内で`cargo loco doctor`を実行して、アプリと環境が新しいバージョンと互換性があることを確認します

いつものように、何か問題が発生した場合は、[issueを開いて](https://github.com/loco-rs/loco/issues)ヘルプを求めてください。

## Locoの主要な依存関係

Locoは優れたライブラリの上に構築されています。Locoの新しいリリースでは、それらのバージョンと個別のchangelogに注意を払うことが賢明です。

主要なものは以下の通りです：

- [SeaORM](https://www.sea-ql.org/SeaORM), [CHANGELOG](https://github.com/SeaQL/sea-orm/blob/master/CHANGELOG.md)
- [Axum](https://github.com/tokio-rs/axum), [CHANGELOG](https://github.com/tokio-rs/axum/blob/main/axum/CHANGELOG.md)

## 0.15.xから0.16.xへのアップグレード

### `Hooks`トレイトの`init_logger`で`Config`の代わりに`AppContext`を使用

PR: [#1418](https://github.com/loco-rs/loco/pull/1418)

独自のロギングを設定するために`Hooks`トレイトの`impl`で`init_logger`の実装を提供している場合、以下の変更を行う必要があります：

```diff
- fn init_logger(config: &config::Config, env: &Environment) -> Result<bool> {
+ fn init_logger(ctx: &AppContext) -> Result<bool> {
```

`init_logger`実装内で`config`を使用するコードは、`ctx.config`を通じてアクセスできます。さらに、新しい`shared_store`など、`AppContext`内の他のすべてのものにもアクセスできます。`env`パラメータも削除され、`AppContext`から`ctx.environment`としてアクセス可能です。

### validatorの組み込みメール検証への切り替え

PR: [#1359](https://github.com/loco-rs/loco/pull/1359)

locoのカスタムメールバリデーターから、`validator`の組み込みメールバリデーターに切り替えます。

```diff
- #[validate(custom (function = "validation::is_valid_email"))]
+ #[validate(email(message = "invalid email"))]
  pub email: String,
```

### ジョブシステム

PR: [#1384](https://github.com/loco-rs/loco/pull/1384)
PR: [#1396](https://github.com/loco-rs/loco/pull/1396)

バックグラウンドジョブシステムに2つの大きな変更が加えられました：

1. RedisプロバイダーはSidekiq互換ではなくなり、カスタム実装を使用します
2. すべてのプロバイダー（Redis、PostgreSQL、SQLite）がタグベースのジョブフィルタリングをサポートするようになりました

#### 変更内容

##### Sidekiq互換性の削除

Redisバックグラウンドジョブシステムは完全にリファクタリングされ、Sidekiq互換の実装が新しいカスタム実装に置き換えられました。これにより柔軟性が向上し、パフォーマンスが改善されますが、以下の点に注意が必要です：

- 古いLocoバージョン（0.16以前）からプッシュされたジョブは認識されず、処理されません
- Redisのデータ構造が完全に変更されました
- 既存のキューに入っているジョブの自動移行パスはありません

##### ジョブフィルタリングの追加

すべてのバックグラウンドワーカープロバイダーに新しいタグベースのジョブフィルタリングシステムが追加されました：

- ワーカーが処理したいタグを指定できるようになりました
- ジョブはエンキュー時にタグ付けできます
- タグのないワーカーはタグ付けされていないジョブのみを処理し、タグ付きワーカーは一致するタグのジョブを処理します
- すべてのプロバイダーで同じAPIが使用されます

#### アップグレード方法

新しいジョブシステムにアップグレードするには：

1. **既存のジョブを処理する**：

   - アップグレード前に、キュー内のすべてのジョブが処理/完了していることを確認してください

2. **古いデータをクリーンアップする**：

   - Redis：ジョブに使用されているRedisデータベースをフラッシュします（`FLUSHDB`コマンド）
   - PostgreSQL：ジョブキューテーブルを削除します
   - SQLite：ジョブキューテーブルを削除します

3. **Locoを更新する**：
   - Loco 0.16+に更新します
   - Locoは初回実行時に正しいスキーマで新しいジョブテーブルを自動的に作成します

### ジェネリックキャッシュ

PR: [#1385](https://github.com/loco-rs/loco/pull/1385)

キャッシュAPIが、文字列だけでなく、シリアライズ可能な任意の型の保存と取得をサポートするようにリファクタリングされました。これは破壊的変更であり、コードの更新が必要です：

#### 破壊的変更：

1. **型パラメータが必要**：すべてのキャッシュメソッドで明示的な型パラメータが必要になりました
2. **メソッドシグネチャ**：ジェネリクスをサポートするため、一部のメソッドシグネチャが変更されました
3. **オブジェクトのシリアライズ**：保存する型はserdeの`Serialize`と`Deserialize`を実装する必要があります

#### 移行ガイド：

**以前：**

```rust
// キャッシュから文字列値を取得
let value = cache.get("key").await?;

// コールバックで挿入または取得
let value = app_ctx.cache.get_or_insert("key", async {
    Ok("value".to_string())
}).await.unwrap();

// 有効期限付きで挿入または取得
let value = app_ctx.cache.get_or_insert_with_expiry("key", Duration::from_secs(300), async {
    Ok("value".to_string())
}).await.unwrap();
```

**以後：**

```rust
// キャッシュから文字列値を取得 - 型を指定
let value = cache.get::<String>("key").await?;

// シリアライズ可能な任意の型で直接挿入
cache.insert("key", &"value".to_string()).await?;

// コールバックで挿入または取得 - 戻り値の型を指定
let value = app_ctx.cache.get_or_insert::<String, _>("key", async {
    Ok("value".to_string())
}).await.unwrap();

// 複雑な型を保存
#[derive(Serialize, Deserialize)]
struct User {
    name: String,
    age: u32,
}

let user = app_ctx.cache.get_or_insert_with_expiry::<User, _>(
    "user:1",
    Duration::from_secs(300),
    async {
        Ok(User { name: "Alice".to_string(), age: 30 })
    }
).await.unwrap();
```

#### カスタム型の実装：

カスタム型をキャッシュで使用するには、`Serialize`と`Deserialize`を実装していることを確認してください：

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
struct MyType {
    // fields...
}
```

### 認証エラー処理

認証エラー処理が改善され、実際の認証失敗とシステムエラーをより適切に区別できるようになりました：

1. **システムエラーは500を返すように**：認証中のデータベースエラーは、Unauthorized (401)ではなくInternal Server Error (500)を返すようになりました
2. **エラーログの改善**：認証エラーは`tracing::error`を使用して詳細なメッセージでログに記録されるようになりました
3. **メッセージの変更**：一般的なエラーメッセージが"other error: '{e}'"から"could not authorize"に更新されました

#### 移行ガイド：

認証中のデータベースエラーが401ステータスコードを返すことに依存するコードがある場合、エラー処理を更新する必要があります。データベース接続の問題で401を期待するコードは、500レスポンスも処理する必要があります。

クライアントアプリケーションは、認証失敗時に401と500の両方のステータスコードを処理する準備が必要です。401は認証の問題を示し、500はシステムエラーを示します。

## 0.14.xから0.15.xへのアップグレード

### validatorクレートのアップグレード

PR: [#1199](https://github.com/loco-rs/loco/pull/1199)

`Cargo.toml`の`validator`クレートのバージョンを更新します：

変更前

```
validator = { version = "0.19" }
```

変更後

```
validator = { version = "0.20" }
```

### ユーザークレーム

PR: [#1159](https://github.com/loco-rs/loco/pull/1159)

- カスタムユーザークレームのフラット化された(デ)シリアライゼーション：
  `UserClaims`の`claims`フィールドが`Option<Value>`から`Map<String, Value>`に変更されました。

- `generate_token`関数での必須マップ値：
  `generate_token`を呼び出す際、`Map<String, Value>`引数が必須になりました。カスタムクレームを使用しない場合は、空のマップ（`serde_json::Map::new()`）を渡してください。

- generate_tokenシグネチャの更新：
  `generate_token`関数は、`expiration`を参照ではなく値として受け取るようになりました。

### ページネーションレスポンス

PR: [#1197](https://github.com/loco-rs/loco/pull/1197)

ページネーションレスポンスに`total_items`フィールドが含まれるようになり、利用可能なアイテムの総数を提供します。

```JSON
{"results":[],"pagination":{"page":0,"page_size":0,"total_pages":0,"total_items":0}}
```

### マイグレーションでの明示的なid

PR: [#1268](https://github.com/loco-rs/loco/pull/1268)

`create_table`を使用するマイグレーションには`("id", ColType::PkAuto)`が必要になりました。新しいマイグレーションには自動的にこのフィールドが追加されます。

```diff
  async fn up(&self, m: &SchemaManager) -> Result<(), DbErr> {
        create_table(m, "movies",
            &[
+           ("id", ColType::PkAuto),
            ("title", ColType::StringNull),
            ],
            &[
            ("user", ""),
            ]
        ).await
    }
```

## 0.13.xから0.14.xへのアップグレード

### Axum 0.7から0.8へのアップグレード

PR: [#1130](https://github.com/loco-rs/loco/pull/1130)
Axum 0.8へのアップグレードには破壊的変更が含まれています。詳細については、[アナウンスメント](https://tokio.rs/blog/2025-01-01-announcing-axum-0-8-0)を参照してください。

#### アップグレード手順

- `Cargo.toml`で、Axumのバージョンを`0.7.5`から`0.8.1`に更新します。
- `use axum::async_trait;`を`use async_trait::async_trait;`に置き換えます。詳細については[こちら](https://tokio.rs/blog/2025-01-01-announcing-axum-0-8-0#async_trait-removal)を参照してください。
- URLパラメータの構文が変更されました。更新された構文については[このセクション](https://tokio.rs/blog/2025-01-01-announcing-axum-0-8-0#path-parameter-syntax-changes)を参照してください。新しいパスパラメータのフォーマット：
  パスパラメータの構文が`/:single`と`/*many`から`/{single}`と`/{*many}`に変更されました。

### `boot`関数フックの拡張

PR: [#1143](https://github.com/loco-rs/loco/pull/1143)

`boot`フック関数は追加のConfigパラメータを受け入れるようになりました。関数シグネチャは以下のように変更されました：

変更前

```rust
async fn boot(mode: StartMode, environment: &Environment) -> Result<BootResult> {
     create_app::<Self, Migrator>(mode, environment).await
}
```

変更後：

```rust
async fn boot(mode: StartMode, environment: &Environment, config: Config) -> Result<BootResult> {
     create_app::<Self, Migrator>(mode, environment, config).await
}
```

必要に応じて`Config`型をインポートしてください。

### validatorクレートのアップグレード

PR: [#993](https://github.com/loco-rs/loco/pull/993)

`Cargo.toml`の`validator`クレートのバージョンを更新します：

変更前

```
validator = { version = "0.18" }
```

変更後

```
validator = { version = "0.19" }
```

### Extend truncate and seed hooks

PR: [#1158](https://github.com/loco-rs/loco/pull/1158)

The `truncate` and `seed` functions now receive `AppContext` instead of `DatabaseConnection` as their argument.

From

```rust
async fn truncate(db: &DatabaseConnection) -> Result<()> {}
async fn seed(db: &DatabaseConnection, base: &Path) -> Result<()> {}
```

To

```rust
async fn truncate(ctx: &AppContext) -> Result<()> {}
async fn seed(_ctx: &AppContext, base: &Path) -> Result<()> {}
```

Impact on Testing:

Testing code involving the seed function must also be updated accordingly.

from:

```rust
async fn load_page() {
    request::<App, _, _>(|request, ctx| async move {
        seed::<App>(&ctx.db).await.unwrap();
        ...
    })
    .await;
}
```

to

```rust
async fn load_page() {
    request::<App, _, _>(|request, ctx| async move {
        seed::<App>(&ctx).await.unwrap();
        ...
    })
    .await;
}
```
