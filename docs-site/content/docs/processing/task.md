+++
title = "タスク"
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

`Loco`のタスクは、アプリケーションの特定の側面を処理するためのアドホックな機能として機能します。データの修正、メール送信、ユーザーの削除、顧客注文の更新など、各シナリオに対して専用のタスクを作成することで、柔軟で効率的なソリューションを提供します。タスクは手動で実行することも、[タスクをスケジュール](@/docs/processing/scheduler.md)することもできます。

タスクを作成することには以下のような利点があります：
- **手作業の自動化：** タスクは手動プロセスを自動化し、反復的なアクションを効率化します。
- **既存コンポーネントの活用：** タスク内でアプリのモデル、ライブラリ、既存のロジックを活用できます。
- **UI開発の不要：** タスクはユーザーインターフェースの構築を必要とせず、バックエンドの操作のみに焦点を当てます。
- **UI自動化の可能性：** 必要に応じて、Jenkinsなどのジョブ実行ツールと統合することで、タスクをUIで自動化できます。

各タスクは、CLIのyargs解析された出力を利用して、コマンドライン引数をフラグに解析するように設計されています。

## CLIジェネレーターを使用したタスクの作成

`Loco`は、プロジェクトに接続されたスタータータスクの作成を簡素化する便利なコードジェネレーターを提供します。タスクを生成するには、以下のコマンドを使用します：

タスクの生成：

<!-- <snip id="generate-task-help-command" inject_from="yaml" action="exec" template="sh"> -->
```sh
Generate a Task based on the given name

Usage: demo_app-cli generate task [OPTIONS] <NAME>

Arguments:
  <NAME>  Name of the thing to generate

Options:
  -e, --environment <ENVIRONMENT>  Specify the environment [default: development]
  -h, --help                       Print help
  -V, --version                    Print version
```
<!-- </snip> -->

## タスクの実行

前のステップで作成したタスクを実行するには、以下のコマンドを使用します：

<!-- <snip id="run-task-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco task <TASK_NAME>
```
<!-- </snip> -->

### パラメーター付きでタスクを実行

タスクにパラメーターを渡すには、コマンドにkey:valueのリストを追加します
```sh
[PARAMS]...  Task params (e.g. <`my_task`> foo:bar baz:qux)`
```
```sh
cargo loco task <TASK_NAME> [PARAMS]...
```

その後、タスクの`run`メソッドで[`cli_arg`](https://docs.rs/loco-rs/latest/loco_rs/task/struct.Vars.html#method.cli_arg)を使用してその値を使用します
```rust
async fn run(&self, app_context: &AppContext, vars: &task::Vars) -> Result<()> {
    let foo = vars.cli_arg("foo");
    Ok(())
}
```

## すべてのタスクの一覧表示

実行されたすべてのタスクのリストを表示するには、以下のコマンドを使用します：

<!-- <snip id="list-tasks-command" inject_from="yaml" template="sh"> -->
```sh
cargo loco task
```
<!-- </snip> -->


## 手動でタスクを作成

`Loco`でタスクを手動で作成したい場合は、以下の手順に従ってください：

#### 1. タスクファイルの作成

`src/tasks`パス下に新しいファイルを作成することから始めます。例として、`example.rs`という名前のファイルを作成しましょう：

<!-- <snip id="task-code-example" inject_from="code" template="rust"> -->
```rust
use loco_rs::prelude::*;

pub struct Foo;
#[async_trait]
impl Task for Foo {
    fn task(&self) -> TaskInfo {
        TaskInfo {
            name: "foo".to_string(),
            detail: "run foo task".to_string(),
        }
    }
    async fn run(&self, _app_context: &AppContext, _vars: &task::Vars) -> Result<()> {
        Ok(())
    }
}
```
<!-- </snip> -->

#### 2. mod.rsでファイルを読み込む

次に、`src/tasks`フォルダー内の`mod.rs`ファイルで、新しく作成したタスクファイルを読み込むようにしてください。

#### 3. Appフックでタスクを登録

Appフックの実装（例：App構造体）で、register_tasks関数でタスクを登録します：

```rust
// src/app.rs

pub struct App;
#[async_trait]
impl Hooks for App {
    ...

    fn register_tasks(tasks: &mut Tasks) {
        tasks.register(tasks::example::ExampleTask);
    }

    ...
}
```

これらの手順により、ExampleTaskなどの手動で作成したタスクがLocoのタスク管理システムに統合されます。
