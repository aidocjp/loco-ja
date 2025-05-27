+++
title = "スケジューラー"
description = ""
date = 2024-11-09T18:10:00+00:00
updated = 2024-11-09T18:10:00+00:00
draft = false
weight = 5
sort_by = "weight"
template = "docs/page.html"

[extra]
lead = ""
toc = true
top = false
flair =[]
+++


Locoは従来の面倒な`crontab`システムをシンプルにし、cronジョブのスケジューリングをより簡単でエレガントにします。スケジューラージョブは、シェルスクリプトコマンドを実行するか、登録済みの[タスク](@/docs/processing/task.md)を実行できます。


## セットアップ
スケジューラージョブは、専用のYAMLスケジューラー設定ファイル、または環境YAMLファイルの一部として設定できます。


### 1. 専用ファイル
専用ファイルを使用すると、すべてのスケジューラージョブを設定するための集中管理された場所が提供され、管理とメンテナンスが容易になります。Locoジェネレーターコマンドを使用してテンプレートファイルを生成することから始められます：

```sh
cargo loco generate scheduler
```

このコマンドは`config`フォルダー下に`scheduler.yaml`ファイルを作成します。その後、このファイル内でジョブを設定できます。

### 2. 環境設定ファイル
環境のYAML設定ファイルにスケジューラーセクションを追加することで、環境ごとにスケジューラージョブを設定することもできます：

<!-- <snip id="configuration-scheduler" inject_from="code" template="yaml"> -->
```yaml
scheduler:
  # Location of shipping the command stdout and stderr.
  output: stdout
  # A list of jobs to be scheduled.
  jobs:
    # The name of the job.
    write_content:
      # by default false meaning executing the the run value as a task. if true execute the run value as shell command
      shell: true
      # command to run
      run: "echo loco >> ./scheduler.txt"
      # The cron expression that defines the job's schedule. 
      schedule: run every 1 second
      output: silent
      tags: ['base', 'infra']

    run_task:
      run: "foo"
      schedule: "at 10:00 am"
      run_on_start: true

    list_if_users:
      run: "user_report"
      shell: true
      schedule: "* 2 * * * *"
      tags: ['base', 'users']
```
<!-- </snip> -->


## スケジューラー設定

スケジューラー設定は以下の要素で構成されています：

* `scheduler.output`（オプション）：すべてのジョブのデフォルト出力場所を設定します。
    * `stdout:`コンソールに出力（デフォルト）。
    * `silent:`すべての出力を抑制。
* `scheduler.jobs:`スケジュールされるジョブのオブジェクト。オブジェクトキーはジョブ名を表します。各ジョブには以下があります：
    * `schedule`：ジョブのスケジュールを定義するcron式。 
        cronは英語からcron構文に変換されるか、cron構文そのものを受け取ります。 

        ##### ***英語からcronへ***
        * 例：
        * every 15 seconds
        * run every minute
        * fire every day at 4:00 pm
        * at 10:00 am
        * run at midnight on the 1st and 15th of the month
        * On Sunday at 12:00
        * 7pm every Thursday
        * midnight on Tuesdays

        ##### ***Cron構文形式：***
        cronjobはUTCベースである必要があります
        ```sh
        sec   min   hour   day of month   month   day of week   year
        *     *     *      *              *       *             *
        ```
  * `run_on_start`：デフォルトは`false`。`true`に設定すると、ジョブはスケジューラーの開始時にも実行されます。
    * `shell`：デフォルトは`false`で、`run`値をタスクとして実行することを意味します。`true`の場合、`run`値をシェルコマンドとして実行します
    * `run`：実行するCronjobコマンド。 
        * `Task:`タスク名（変数付き 例：`[TASK_NAME] KEY:VAl`。タスク引数については[こちら](@/docs/processing/task.md)を参照）。`shell`フィールドはfalseである必要があります。
        * `Shell:`シェルコマンドを実行（例：`"echo loco >> ./scheduler.txt"`）。`shell`フィールドはtrueである必要があります。
    * `tags`（オプション）：ジョブを分類および管理するためのタグのリスト。
    * `output`（オプション）：このジョブのグローバル`scheduler.output`を上書きします。


## 設定の検証
ジョブをセットアップした後、設定が正しいことを確認するために検証できます。

### 1. 専用ファイルを使用する場合：
スケジューラーファイルからジョブをリストするには、以下のコマンドを実行します：
<!-- <snip id="scheduler-list-from-file-command" inject_from="yaml"  template="sh"> -->
```sh
cargo loco scheduler --config config/scheduler.yaml --list
```
<!-- </snip> -->

### 2. 環境ベースの設定を使用する場合：
環境設定からジョブをリストするには、以下を実行します：
<!-- <snip id="scheduler-list-from-env-setting-command" inject_from="yaml"  template="sh"> -->
```sh
LOCO_ENV=production cargo loco scheduler --list
```
<!-- </snip> -->


## スケジューラーの実行
設定が検証されたら、`--list`フラグを削除してスケジューラーの実行を開始できます。スケジューラーは、シャットダウンシグナルを受信するまで、スケジュールに基づいてジョブを継続的に実行します。シグナルを受信すると、実行中のすべてのタスクを正常に終了し、安全にシャットダウンします。

### 重要な注意事項：
* ジョブが実行されると、`Loco`は新しいプロセスでそれを生成し、すべての環境変数が新しいジョブプロセスに伝播されます。
* タスクの場合、`--environment`フラグを使用するか、`LOCO_ENV`環境変数を設定して、有効な環境でスケジューラーを実行するようにしてください。これにより、タスクに対して正しい環境と設定が読み込まれることが保証されます。
* タスク設定のvarsオブジェクトを使用して、タスクに変数を渡すことができます。


## 名前による単一スケジュールジョブの実行
特定のスケジューラージョブをその名前で実行するには、--nameフラグを使用します。これにより、指定された名前の単一ジョブが実行されます。
<!-- <snip id="scheduler-run-job-by-name-command" inject_from="yaml"  template="sh"> -->
```sh
LOCO_ENV=production cargo loco scheduler --name 'JOB_NAME'
```
<!-- </snip> -->

このコマンドは、scheduler.yamlファイル内の`"Run command"`という名前のジョブを検索して実行します。

## タグによるスケジュールジョブの実行
同じタグを共有する複数のジョブを実行することもできます。タグは関連するジョブをグループ化するのに便利です。例えば、データベースのクリーンアップ、キャッシュの無効化、ログのローテーションなど、さまざまな種類のメンテナンスタスクを実行する複数のジョブがあり、それらを一緒に実行したい場合があります。これらに`maintenance`のような同じタグを割り当てることで、すべてを一度に実行できます。
<!-- <snip id="scheduler-run-job-by-tag-command" inject_from="yaml"  template="sh"> -->
```sh
LOCO_ENV=production cargo loco scheduler --tag 'maintenance'
```
<!-- </snip> -->


このコマンドは、`maintenance`でタグ付けされたすべてのジョブを実行し、関連するすべてのジョブが一度に実行されることを保証します。
