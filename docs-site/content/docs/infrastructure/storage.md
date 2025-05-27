+++
title = "Storage"
description = ""
date = 2024-02-07T08:00:00+00:00
updated = 2024-02-07T08:00:00+00:00
draft = false
weight = 1
sort_by = "weight"
template = "docs/page.html"

[extra]
lead = ""
toc = true
top = false
flair =[]
+++

Loco Storageでは、複数の操作を通じてファイルの操作を容易にします。ストレージはメモリ内、ディスク上、またはAWS S3、GCP、Azureなどのクラウドサービスを使用できます。

Locoは、シンプルなストレージ操作と、異なる障害モードを持つデータのミラーリングやバックアップ戦略などの高度な機能をサポートしています。

デフォルトでは、メモリ内およびディスクストレージがすぐに使用できます。クラウドプロバイダーと連携するには、以下の機能を指定する必要があります：
- `storage_aws_s3`
- `storage_azure`
- `storage_gcp`
- `all_storage`

デフォルトでlocoは`Null`プロバイダーを初期化します。これは、ストレージに関する作業がエラーを返すことを意味します。 

## セットアップ

`app.rs`ファイルにフックとして`after_context`関数を追加し、`loco_rs`から`storage`モジュールをインポートします。

```rust
use loco_rs::storage;

async fn after_context(ctx: AppContext) -> Result<AppContext> {
    Ok(ctx)
}
```

このフックは、次のセクションで説明するすべてのストレージ設定を保持するStorageインスタンスを返します。このStorageインスタンスは、アプリケーションコンテキストの一部として保存され、コントローラー、エンドポイント、タスクワーカーなどで利用できます。

## 用語集
|          |   |
| -        | - |
| `StorageDriver` | ストレージを実行するトレイト実装  |
| `Storage`| 1つ以上のストレージドライバーを管理するための抽象化実装 |
| `Strategy`| ミラーやバックアップなど、ストレージのさまざまな戦略を実装するトレイト |
| `FailureMode`| 各戦略内で実装され、失敗時の操作の処理方法を決定する |

### ストレージの初期化

ストレージは、単一のドライバーまたは複数のドライバーで構成できます。

#### シングルストア

この例では、メモリ内ドライバーを初期化し、single関数で新しいストレージを作成します。

```rust
use loco_rs::storage;

async fn after_context(ctx: AppContext) -> Result<AppContext> {
    Ok(AppContext {
        storage: Storage::single(storage::drivers::mem::new()).into(),
        ..ctx
    })
}
```

### 複数のドライバー

高度な使用法として、複数のドライバーを設定し、組み込みのスマートな戦略を適用できます。各戦略には、処理方法を決定できる独自の障害モードのセットがあります。

複数のドライバーの作成：

```rust
use crate::storage::{drivers, Storage};

let aws_1 = drivers::aws::new("users");
let azure = drivers::azure::new("users");
let aws_2 = drivers::aws::new("users-mirror");
```

#### ミラー戦略：
ミラーサービスを定義することで、複数のサービスを同期させることができます。ミラーサービスは、アップロード、削除、名前変更、コピーを2つ以上の従属サービス全体に**複製**します。ダウンロード動作は冗長的にデータを取得します。つまり、プライマリからのファイル取得が失敗した場合、セカンダリで見つかった最初のファイルが返されます。

#### 動作

3つのストアインスタンスを作成した後、ミラー戦略インスタンスを作成し、障害モードを定義する必要があります。ミラー戦略は、プライマリストアとセカンダリストアのリスト、および障害モードオプションを期待します：
- `MirrorAll`：すべてのセカンダリストレージが成功する必要があります。1つが失敗しても、操作は残りに継続しますが、エラーを返します。
- `AllowMirrorFailure`：1つ以上のミラー操作が失敗しても、操作はエラーを返しません。

障害モードは、アップロード、削除、移動、コピーに関連します。

例：
```rust

// 名前でプライマリストアとセカンダリストアを設定して、ミラー戦略を定義します。
let strategy = Box::new(MirrorStrategy::new(
    "store_1",
    Some(vec!["store_2".to_string(), "store_3".to_string()]),
    FailureMode::MirrorAll,
)) as Box<dyn StorageStrategy>;

// ストアマッピングと戦略を使用してストレージを作成します。
 let storage = Storage::new(
    BTreeMap::from([
        ("store_1".to_string(), aws_1),
        ("store_2".to_string(), azure),
        ("store_3".to_string(), aws_2),
    ]),
    strategy.into(),
);
```

### バックアップ戦略：

複数のストレージ間で操作をバックアップし、障害モードポリシーを制御できます。

3つのストアインスタンスを作成した後、バックアップ戦略インスタンスを作成し、障害モードを定義する必要があります。バックアップ戦略は、プライマリストアとセカンダリストアのリスト、および障害モードオプションを期待します：
- `BackupAll`：すべてのセカンダリストレージが成功する必要があります。1つが失敗しても、操作は残りに継続しますが、エラーを返します。
- `AllowBackupFailure`：1つ以上のバックアップ操作が失敗しても、操作はエラーを返しません。
- `AtLeastOneFailure`：少なくとも1つの操作が成功する必要があります。
- `CountFailure`：指定された数のバックアップが成功する必要があります。

障害モードは、アップロード、削除、移動、コピーに関連します。ダウンロードは常にプライマリからファイルを取得します。

例：
```rust

// 名前でプライマリストアとセカンダリストアを設定して、バックアップ戦略を定義します。
let strategy: Box<dyn StorageStrategy> = Box::new(BackupStrategy::new(
    "store_1",
    Some(vec!["store_2".to_string(), "store_3".to_string()]),
    FailureMode::AllowBackupFailure,
)) as Box<dyn StorageStrategy>;

let storage = Storage::new(
    BTreeMap::from([
        ("store_1".to_string(), store_1),
        ("store_2".to_string(), store_2),
        ("store_3".to_string(), store_3),
    ]),
    strategy.into(),
);
```

## 独自の戦略を作成する

特定の戦略がある場合は、StorageStrategyを実装し、すべてのストア機能を実装することで簡単に作成できます。

## コントローラーでの使用

この例に従ってください。axumクレートで`multipart`機能を有効にしてください。

```rust
async fn upload_file(
    State(ctx): State<AppContext>,
    mut multipart: Multipart,
) -> Result<Response> {
    let mut file = None;
    while let Some(field) = multipart.next_field().await.map_err(|err| {
        tracing::error!(error = ?err,"could not readd multipart");
        Error::BadRequest("could not readd multipart".into())
    })? {
        let file_name = match field.file_name() {
            Some(file_name) => file_name.to_string(),
            _ => return Err(Error::BadRequest("file name not found".into())),
        };

        let content = field.bytes().await.map_err(|err| {
            tracing::error!(error = ?err,"could not readd bytes");
            Error::BadRequest("could not readd bytes".into())
        })?;

        let path = PathBuf::from("folder").join(file_name);
        ctx.storage.as_ref().upload(path.as_path(), &content).await?;

        file = Some(path);
    }

    file.map_or_else(not_found, |path| {
        format::json(views::upload::Response::new(path.as_path()))
    })
}
```
# テスト

コントローラーでファイルストレージをテストするには、この例に従ってください：

```rust
use loco_rs::testing::prelude::*;

#[tokio::test]
#[serial]
async fn can_register() {
    request::<App, _, _>(|request, ctx| async move {
        let file_content = "loco file upload";
        let file_part = Part::bytes(file_content.as_bytes()).file_name("loco.txt");

        let multipart_form = MultipartForm::new().add_part("file", file_part);

        let response = request.post("/upload/file").multipart(multipart_form).await;

        response.assert_status_ok();

        let res: views::upload::Response = serde_json::from_str(&response.text()).unwrap();

        let stored_file: String = ctx.storage.as_ref().download(&res.path).await.unwrap();

        assert_eq!(stored_file, file_content);
    })
    .await;
}
```

