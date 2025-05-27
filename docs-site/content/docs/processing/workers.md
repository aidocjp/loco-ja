+++
title = "Workers"
description = ""
date = 2021-05-01T18:10:00+00:00
updated = 2021-05-01T18:10:00+00:00
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

Locoは以下のバックグラウンドジョブのオプションを提供します：

- Redisベース
- Postgresベース
- SQLiteベース
- Tokio-asyncベース（同一プロセス、イベント駆動スレッドベースのバックグラウンドジョブ）

Railsの_ActiveJob_と同様に、実際のバックグラウンドキューの実装を意識することなくジョブをエンキューして実行できます。そのため、設定を変更するだけでコードを変更することなく切り替えることができます。

## 非同期 vs キュー

新しいアプリを生成した際、ワーカーにデフォルトの`async`設定を選択したかもしれません。これは、ワーカーがTokioの非同期プール内でジョブをスピンオフし、同じ実行中のサーバー内で適切なバックグラウンドプロセスを提供することを意味します。

サーバー間で負荷を分散するために、キューに支えられた別のプロセスでジョブを実行するよう設定したい場合があります。

まず、`BackgroundQueue`に切り替えます：

```yaml
# ワーカーの設定
workers:
  # specifies the worker mode. Options:
  #   - BackgroundQueue - Workers operate asynchronously in the background, processing queued.
  #   - ForegroundBlocking - Workers operate in the foreground and block until tasks are completed.
  #   - BackgroundAsync - Workers operate asynchronously in the background, processing tasks with async capabilities.
  mode: BackgroundQueue
```

次に、Redisベースのキューバックエンドを設定します：

```yaml
queue:
  kind: Redis
  # Redis connection URI.
  uri: "{{ get_env(name="REDIS_URL", default="redis://127.0.0.1") }}"
  # Dangerously flush all data.
  dangerously_flush: false
  # represents the number of tasks a worker can handle simultaneously.
  num_workers: 2
```

またはPostgresベースのキューバックエンド：

```yaml
queue:
  kind: Postgres
  # Postgres Queue connection URI.
  uri: "{{ get_env(name="PGQ_URL", default="postgres://localhost:5432/mydb") }}"
  # Dangerously flush all data.
  dangerously_flush: false
  # represents the number of tasks a worker can handle simultaneously.
  num_workers: 2
```

またはSQLiteベースのキューバックエンド：

```yaml
queue:
  kind: Sqlite
  # SQLite Queue connection URI.
  uri: "{{ get_env(name="SQLTQ_URL", default="sqlite://loco_development.sqlite?
  mode=rwc") }}"
  # Dangerously flush all data.
  dangerously_flush: false
  # represents the number of tasks a worker can handle simultaneously.
  num_workers: 2
```

## ワーカープロセスの実行

バックグラウンドワーカーに選択した設定に応じて、2つの方法で実行できます：

```
Usage: demo_app start [OPTIONS]

Options:
  -w, --worker [<WORKER>...]       Start worker. Optionally provide tags to run specific jobs (e.g. --worker=tag1,tag2)
  -s, --server-and-worker          start same-process server and worker
```

実際のRedisキューを設定し、バックグラウンドジョブだけを実行するプロセスが必要な場合は`--worker`を選択します。サーバーごとに単一のプロセスを使用できます。この場合、メインのWebまたはAPIサーバーは単に`cargo loco start`を使用して実行できます。

```sh
$ cargo loco start --worker # starts a standalone worker job executing process
$ cargo loco start # starts a standalone API service or Web server, no workers
```

`async`バックグラウンドワーカーを設定した場合は`-s`を選択し、ジョブは現在実行中のサーバープロセスの一部として実行されます。

例えば、`--server-and-worker`を実行する場合：

```sh
$ cargo loco start --server-and-worker # both API service and workers will execute
```

### ワーカータグフィルタリング

Locoはタグベースのジョブフィルタリングをサポートしており、特定のタイプのジョブのみを処理する専門的なワーカーを作成できます。これは、ワークロードを分散したり、リソース集約的なタスク用の専用ワーカーを作成したりする場合に特に便利です。

ワーカーを起動する際、処理すべきタグを指定できます：

```sh
# タグを持たないジョブのみを処理するワーカーを開始
$ cargo loco start --worker

# "email"タグを持つジョブのみを処理するワーカーを開始
$ cargo loco start --worker email

# "report"または"analytics"タグのいずれかを持つジョブを処理するワーカーを開始
$ cargo loco start --worker report,analytics
```

タグベースの処理に関する重要な注意事項：

1. タグを持たないワーカー（`cargo loco start --worker`）は、タグを持たないジョブのみを処理します
2. タグを持つワーカーは、少なくとも1つの一致するタグを持つジョブのみを処理します
3. `--all`および`--server-and-worker`モードはタグによるフィルタリングをサポートしておらず、タグなしのジョブのみを処理します
4. タグは大文字小文字を区別します

## コードでバックグラウンドジョブを作成

ワーカーを使用するには、主にキューにジョブを追加することを考えます。ワーカーを`use`して後で実行します：

```rust
    // .. in your controller ..
    DownloadWorker::perform_later(
        &ctx,
        DownloadWorkerArgs {
            user_guid: "foo".to_string(),
        },
    )
    .await
```

RailsとRubyとは異なり、Rustでは_強型付け_されたジョブ引数を使用でき、これはシリアライズされてキューにプッシュされます。

### ジョブへのタグ割り当て

ジョブをエンキューする際、オプションでタグを割り当てることができます。ジョブは、そのタグの少なくとも1つと一致するワーカーによってのみ処理されます：

```rust
    // タグ付きジョブを作成するには、ワーカーでタグを定義します：
    struct DownloadWorker;

    #[async_trait]
    impl BackgroundWorker<DownloadWorkerArgs> for DownloadWorker {
        // このワーカーのタグを定義
        fn tags() -> Vec<String> {
            vec!["download".to_string(), "network".to_string()]
        }

        // ... その他の実装詳細
    }

    // perform_laterを呼び出すと、ジョブは自動的にタグ付けされます
    DownloadWorker::perform_later(&ctx, args).await?;
```

### ワーカーから共有状態を使用

[グローバル状態の使い方](@/docs/the-app/controller.md#global-app-wide-state)を参照してくださいが、一般的には`lazy_static`のようなものを使用して単一の共有状態を使用し、ワーカーから単純に参照します。

この状態がシリアライズ可能な場合は、`WorkerArgs`を通じて渡すことを_強く推奨_します。

## 新しいワーカーの作成

ワーカーを追加するということは、_引数_を受け取ってジョブを実行するバックグラウンドジョブロジックをコーディングすることを意味します。また、`loco`にそれを知らせ、グローバルジョブプロセッサに登録する必要があります。

`workers/`にワーカーを追加します：

```rust
#[async_trait]
impl BackgroundWorker<DownloadWorkerArgs> for DownloadWorker {
    fn build(ctx: &AppContext) -> Self {
        Self { ctx: ctx.clone() }
    }

    // オプション：このワーカーのタグを定義
    fn tags() -> Vec<String> {
        vec!["download".to_string()]
    }

    async fn perform(&self, args: DownloadWorkerArgs) -> Result<()> {
        println!("================================================");
        println!("Sending payment report to user {}", args.user_guid);

        // TODO: 実際の作業をここに記述...

        println!("================================================");
        Ok(())
    }
}
```

そして`app.rs`に登録します：

```rust
#[async_trait]
impl Hooks for App {
//..
    async fn connect_workers(ctx: &AppContext, queue: &Queue) -> Result<()> {
        queue.register(DownloadWorker::build(ctx)).await?;
        Ok(())
    }
// ..
}
```

### `BackgroundWorker`トレイト

`BackgroundWorker`トレイトは、Locoでバックグラウンドワーカーを定義するためのコアインターフェースです。いくつかのメソッドを提供します：

- `build(ctx: &AppContext) -> Self`：提供されたアプリケーションコンテキストでワーカーの新しいインスタンスを作成します。
- `perform(&self, args: A) -> Result<()>`：提供された引数でジョブのロジックを実行するメインメソッドです。
- `queue() -> Option<String>`：ワーカーのカスタムキューを指定するオプションメソッド（デフォルトで`None`を返します）。
- `tags() -> Vec<String>`：このワーカーのタグを指定するオプションメソッド（デフォルトで空のベクターを返します）。
- `class_name() -> String`：ワーカーのクラス名を返します（構造体名から自動的に派生されます）。
- `perform_later(ctx: &AppContext, args: A) -> Result<()>`：後で実行するジョブをエンキューする静的メソッドです。

### ワーカーの生成

`loco generate`を使用してワーカーを自動的に追加するには、以下のコマンドを実行します：

```sh
cargo loco generate worker report_worker
```

ワーカージェネレーターは、アプリに関連付けられたワーカーファイルを作成し、テストテンプレートファイルを生成して、ワーカーを検証できるようにします。

## ワーカーの設定

`config/<environment>.yaml`でワーカーモードを指定できます。BackgroundAsyncとBackgroundQueueはノンブロッキング方式でジョブを処理し、ForegroundBlockingはブロッキング方式でジョブを処理します。

BackgroundAsyncとBackgroundQueueの主な違いは、後者がRedis/Postgres/SQLiteを使用してジョブを保存するのに対し、前者はRedis/Postgres/SQLiteを必要とせず、同じプロセス内でインメモリ非同期を使用することです。

```yaml
# ワーカーの設定
workers:
  # specifies the worker mode. Options:
  #   - BackgroundQueue - Workers operate asynchronously in the background, processing queued.
  #   - ForegroundBlocking - Workers operate in the foreground and block until tasks are completed.
  #   - BackgroundAsync - Workers operate asynchronously in the background, processing tasks with async capabilities.
  mode: BackgroundQueue
```

## UIからワーカーを管理

[Loco admin jobプロジェクト](https://github.com/loco-rs/admin-jobs)でジョブキューを管理できます。
![<img style="width:100%; max-width:640px" src="tour.png"/>](https://github.com/loco-rs/admin-jobs/raw/main/media/screenshot.png)

### CLIによるジョブキューの管理

ジョブキュー管理機能は、アプリケーション内のジョブのライフサイクルを処理するための強力で柔軟な方法を提供します。ジョブのキャンセル、クリーンアップ、古いジョブの削除、ジョブ詳細のエクスポート、ジョブのインポートが可能で、効率的で組織化されたジョブ処理を確保します。

## 機能概要

- **ジョブのキャンセル**  
  名前で特定のジョブをキャンセルし、ステータスを`cancelled`に更新する機能を提供します。これは、もはや不要、関連性がない、またはバグが検出された場合に処理を防止したいジョブを停止するのに便利です。
- **ジョブのクリーンアップ**  
  すでに完了またはキャンセルされたジョブの削除を可能にします。これにより、不要なエントリを排除してクリーンで効率的なジョブキューを維持できます。
- **古いジョブのパージ**  
  日数で測定される経過時間に基づいてジョブを削除できます。これは、古い関連性のないジョブを削除してスリムなジョブキューを維持するのに特に便利です。  
  **注意**：`--dump`オプションを使用してジョブ詳細をファイルにエクスポートし、エクスポートされたファイル内のジョブパラメータを手動で変更してから、`import`機能を使用して更新されたジョブをシステムに再導入できます。
- **ジョブ詳細のエクスポート**  
  すべてのジョブの詳細を指定された場所にファイル形式でエクスポートすることをサポートします。この機能は、バックアップ、監査、またはさらなる分析に価値があります。
- **ジョブのインポート**  
  外部ファイルからのジョブのインポートを促進し、システムへの新しいジョブの復元または追加を簡単にします。これにより、外部ジョブデータのアプリケーションワークフローへのシームレスな統合が保証されます。

ジョブ管理コマンドにアクセスするには、以下のCLI構造を使用します：

<!-- <snip id="jobs-help-command" inject_from="yaml" action="exec" template="sh"> -->

```sh
Managing jobs queue

Usage: demo_app-cli jobs [OPTIONS] <COMMAND>

Commands:
  cancel  Cancels jobs with the specified names, setting their status to `cancelled`
  tidy    Deletes jobs that are either completed or cancelled
  purge   Deletes jobs based on their age in days
  dump    Saves the details of all jobs to files in the specified folder
  import  Imports jobs from a file
  help    Print this message or the help of the given subcommand(s)

Options:
  -e, --environment <ENVIRONMENT>  Specify the environment [default: development]
  -h, --help                       Print help
  -V, --version                    Print version
```

<!-- </snip> -->

## ワーカーのテスト

`Loco`を使用してワーカーのバックグラウンドジョブを簡単にテストできます。ワーカーが`ForegroundBlocking`モードに設定されていることを確認してください。このモードはジョブをブロックし、同期的に実行されることを保証します。ワーカーをテストする際、テストはワーカーが完了するまで待機し、ワーカーが意図したタスクを達成したかどうかを検証できます。

すべてのワーカーテストを一箇所にまとめるため、`tests/workers`ディレクトリにテストを実装することをお勧めします。

さらに、自動的にテストを作成するワーカージェネレーターを活用でき、ライブラリでのテスト設定にかかる時間を節約できます。

テストの構造化方法の例は以下の通りです：

```rust
use loco_rs::testing::prelude::*;

#[tokio::test]
#[serial]
async fn test_run_report_worker_worker() {
    // テスト環境をセットアップ
    let boot = boot_test::<App, Migrator>().await.unwrap();

    // ワーカーを'ForegroundBlocking'モードで実行し、非同期実行を防ぐ
    assert!(
        ReportWorkerWorker::perform_later(&boot.app_context, ReportWorkerWorkerArgs {})
            .await
            .is_ok()
    );

    // ワーカーの実行後に追加のアサート検証を含める
}

```

### `class_name()`の理解

`BackgroundWorker`トレイトの`class_name()`関数は、ジョブキュー内のワーカーの一意識別子を決定するために使用されます。デフォルトでは以下のことを行います：

1. 構造体名を取得（例：`DownloadWorker`）
2. モジュールパスを除去（例：`my_app::workers::DownloadWorker`は単に`DownloadWorker`になります）
3. UpperCamelCase形式に変換

これは重要です。なぜなら、ジョブがエンキューされる際、処理時に適切なワーカーと一致させるための文字列識別子が必要だからです。この関数は自動的にその識別子を生成しますが、カスタムネーミングスキームが必要な場合はオーバーライドできます。

```rust
// class_nameの動作例：
struct download_worker;
impl BackgroundWorker<Args> for download_worker {
    // class_name()は"DownloadWorker"を返します
    // カスタムネーミングが必要でない限り、これをオーバーライドする必要はありません
}
```

そして`app.rs`に登録します：

```rust
#[async_trait]
impl Hooks for App {
//..
    async fn connect_workers(ctx: &AppContext, queue: &Queue) -> Result<()> {
        queue.register(DownloadWorker::build(ctx)).await?;
        Ok(())
    }
// ..
}
```
