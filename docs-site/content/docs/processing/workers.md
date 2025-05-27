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

## Async vs Queue

新しいアプリを生成した際、ワーカーにデフォルトの`async`設定を選択したかもしれません。これは、ワーカーがTokioの非同期プール内でジョブをスピンオフし、同じ実行中のサーバー内で適切なバックグラウンドプロセスを提供することを意味します。

サーバー間で負荷を分散するために、キューに支えられた別のプロセスでジョブを実行するよう設定したい場合があります。

まず、`BackgroundQueue`に切り替えます：

```yaml
# Worker Configuration
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
# Start a worker that only processes jobs with no tags
$ cargo loco start --worker

# Start a worker that only processes jobs with the "email" tag
$ cargo loco start --worker email

# Start a worker that processes jobs with either "report" or "analytics" tags
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
    // To create a job with a tag, define the tags in your worker:
    struct DownloadWorker;

    #[async_trait]
    impl BackgroundWorker<DownloadWorkerArgs> for DownloadWorker {
        // Define tags for this worker
        fn tags() -> Vec<String> {
            vec!["download".to_string(), "network".to_string()]
        }

        // ... other implementation details
    }

    // When you call perform_later, the job will automatically be tagged
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

    // Optional: Define tags for this worker
    fn tags() -> Vec<String> {
        vec!["download".to_string()]
    }

    async fn perform(&self, args: DownloadWorkerArgs) -> Result<()> {
        println!("================================================");
        println!("Sending payment report to user {}", args.user_guid);

        // TODO: Some actual work goes here...

        println!("================================================");
        Ok(())
    }
}
```

And register it in `app.rs`:

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

### The `BackgroundWorker` Trait

The `BackgroundWorker` trait is the core interface for defining background workers in Loco. It provides several methods:

- `build(ctx: &AppContext) -> Self`: Creates a new instance of the worker with the provided application context.
- `perform(&self, args: A) -> Result<()>`: The main method that executes the job's logic with the provided arguments.
- `queue() -> Option<String>`: Optional method to specify a custom queue for the worker (returns `None` by default).
- `tags() -> Vec<String>`: Optional method to specify tags for this worker (returns an empty vector by default).
- `class_name() -> String`: Returns the worker's class name (automatically derived from the struct name).
- `perform_later(ctx: &AppContext, args: A) -> Result<()>`: Static method to enqueue a job to be performed later.

### Generate a Worker

To automatically add a worker using `loco generate`, execute the following command:

```sh
cargo loco generate worker report_worker
```

The worker generator creates a worker file associated with your app and generates a test template file, enabling you to verify your worker.

## Configuring Workers

In your `config/<environment>.yaml` you can specify the worker mode. BackgroundAsync and BackgroundQueue will process jobs in a non-blocking manner, while ForegroundBlocking will process jobs in a blocking manner.

The main difference between BackgroundAsync and BackgroundQueue is that the latter will use Redis/Postgres/SQLite to store the jobs, while the former does not require Redis/Postgres/SQLite and will use async in memory within the same process.

```yaml
# Worker Configuration
workers:
  # specifies the worker mode. Options:
  #   - BackgroundQueue - Workers operate asynchronously in the background, processing queued.
  #   - ForegroundBlocking - Workers operate in the foreground and block until tasks are completed.
  #   - BackgroundAsync - Workers operate asynchronously in the background, processing tasks with async capabilities.
  mode: BackgroundQueue
```

## Manage a Workers From UI

You can manage the jobs queue with the [Loco admin job project](https://github.com/loco-rs/admin-jobs).
![<img style="width:100%; max-width:640px" src="tour.png"/>](https://github.com/loco-rs/admin-jobs/raw/main/media/screenshot.png)

### Managing Job Queues via CLI

The job queue management feature provides a powerful and flexible way to handle the lifecycle of jobs in your application. It allows you to cancel, clean up, remove outdated jobs, export job details, and import jobs, ensuring efficient and organized job processing.

## Features Overview

- **Cancel Jobs**  
  Provides the ability to cancel specific jobs by name, updating their status to `cancelled`. This is useful for stopping jobs that are no longer needed, relevant, or if you want to prevent them from being processed when a bug is detected.
- **Clean Up Jobs**  
  Enables the removal of jobs that have already been completed or cancelled. This helps maintain a clean and efficient job queue by eliminating unnecessary entries.
- **Purge Outdated Jobs**  
  Allows you to delete jobs based on their age, measured in days. This is particularly useful for maintaining a lean job queue by removing older, irrelevant jobs.  
  **Note**: You can use the `--dump` option to export job details to a file, manually modify the job parameters in the exported file, and then use the `import` feature to reintroduce the updated jobs into the system.
- **Export Job Details**  
  Supports exporting the details of all jobs to a specified location in file format. This feature is valuable for backups, audits, or further analysis.
- **Import Jobs**  
  Facilitates importing jobs from external files, making it easy to restore or add new jobs to the system. This ensures seamless integration of external job data into your application's workflow.

To access the job management commands, use the following CLI structure:

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

## Testing a Worker

You can easily test your worker background jobs using `Loco`. Ensure that your worker is set to the `ForegroundBlocking` mode, which blocks the job, ensuring it runs synchronously. When testing the worker, the test will wait until your worker is completed, allowing you to verify if the worker accomplished its intended tasks.

It's recommended to implement tests in the `tests/workers` directory to consolidate all your worker tests in one place.

Additionally, you can leverage the [worker generator](@/docs/processing/workers.md#generate-a-worker), which automatically creates tests, saving you time on configuring tests in the library.

Here's an example of how the test should be structured:

```rust
use loco_rs::testing::prelude::*;

#[tokio::test]
#[serial]
async fn test_run_report_worker_worker() {
    // Set up the test environment
    let boot = boot_test::<App, Migrator>().await.unwrap();

    // Execute the worker in 'ForegroundBlocking' mode, preventing it from running asynchronously
    assert!(
        ReportWorkerWorker::perform_later(&boot.app_context, ReportWorkerWorkerArgs {})
            .await
            .is_ok()
    );

    // Include additional assert validations after the execution of the worker
}

```

### Understanding `class_name()`

The `class_name()` function in the `BackgroundWorker` trait is used to determine the unique identifier for your worker in the job queue. By default, it:

1. Takes the struct name (e.g., `DownloadWorker`)
2. Strips any module paths (e.g., `my_app::workers::DownloadWorker` becomes just `DownloadWorker`)
3. Converts it to UpperCamelCase format

This is important because when a job is enqueued, it needs a string identifier to match with the appropriate worker when it's time for processing. This function automatically generates that identifier for you, but you can override it if you need a custom naming scheme.

```rust
// Example of how class_name works:
struct download_worker;
impl BackgroundWorker<Args> for download_worker {
    // class_name() would return "DownloadWorker"
    // No need to override this unless you need custom naming
}
```

And register it in `app.rs`:

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
