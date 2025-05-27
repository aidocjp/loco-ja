+++
title = "Cache"
description = ""
date = 2024-02-07T08:00:00+00:00
updated = 2025-04-22T08:00:00+00:00
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

`Loco`は頻繁にアクセスされるデータを保存することで、アプリケーションのパフォーマンスを向上させるキャッシュレイヤーを提供します。

## サポートされているキャッシュドライバー

Locoは以下のキャッシュドライバーをすぐに使用できます：

1. **Nullキャッシュ**: 実際には何も保存しない無操作キャッシュ（デフォルト）
2. **インメモリキャッシュ**: `moka`クレートを使用したローカルインメモリキャッシュ
3. **Redisキャッシュ**: Redisを使用した分散キャッシュ

## デフォルトの動作

デフォルトでは、`Loco`は`Null`キャッシュドライバーを初期化します。Nullドライバーはキャッシュインターフェースを実装していますが、実際にはデータを保存しません：

- `get()`操作は常に`None`を返します
- `insert()`、`remove()`などの他の操作は、操作がサポートされていないことを示すメッセージとともにエラーを返します

適切なキャッシュドライバーを設定せずにキャッシュ機能を使用すると、多くの操作でエラーが発生します。本番環境では実際のキャッシュドライバーを設定することをお勧めします。

## キャッシュドライバーの設定

アプリケーションの設定ファイル（例：`config/development.yaml`）で、お好みのキャッシュドライバーを設定できます。

### 設定例

#### Nullキャッシュ（デフォルト）

```yaml
cache:
  kind: Null
```

#### インメモリキャッシュ
`cache_inmem`機能がデフォルトで有効
```yaml
cache:
  kind: InMem
  max_capacity: 33554432 # 32MiB（指定しない場合のデフォルト）
```

#### Redisキャッシュ
`cache_redis`機能を有効にする必要があります
```yaml
cache:
  kind: Redis
  uri: "redis://localhost:6379"
  max_size: 10 # プール内の最大接続数
```

キャッシュ設定が提供されない場合、デフォルトで`Null`キャッシュが使用されます。

## キャッシュの使用

すべてのアイテムは、文字列キーを持つシリアライズされた値としてキャッシュされます。

```rust
use std::time::Duration;
use loco_rs::cache;
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
struct User {
    name: String,
    age: u32,
}

async fn test_cache(ctx: AppContext) -> Result<()> {
    // シンプルな文字列値を挿入
    ctx.cache.insert("string_key", "simple value").await?;

    // 構造化された値を挿入
    let user = User { name: "Alice".to_string(), age: 30 };
    ctx.cache.insert("user:1", &user).await?;

    // 有効期限付きで挿入
    ctx.cache.insert_with_expiry("expiring_key", "temporary value", Duration::from_secs(300)).await?;

    // 文字列値を取得
    let string_value = ctx.cache.get::<String>("string_key").await?;

    // 構造化された値を取得
    let user = ctx.cache.get::<User>("user:1").await?;

    // 値を削除
    ctx.cache.remove("string_key").await?;

    // キーが存在するかチェック
    let exists = ctx.cache.contains_key("user:1").await?;

    // 取得または挿入（存在する場合は取得、存在しない場合は計算して保存）
    let lazy_value = ctx.cache.get_or_insert::<String, _>("lazy_key", async {
        Ok("computed value".to_string())
    }).await?;

    Ok(())
}
```

詳細な例については、[Cache API](https://docs.rs/loco-rs/latest/loco_rs/cache/struct.Cache.html)ドキュメントを参照してください。
