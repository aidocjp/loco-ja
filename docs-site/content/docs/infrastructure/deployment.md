+++
title = "Deployment"
date = 2021-05-01T08:00:00+00:00
updated = 2021-05-01T08:00:00+00:00
draft = false
weight = 3
sort_by = "weight"
template = "docs/page.html"

[extra]
toc = true
top = false
flair =[]
+++

Locoではデプロイメントが非常にシンプルなため、このガイドも非常に短くなっています。**開発時にはほとんどの場合`cargo`を使用していますが**、デプロイ時には**コンパイル済みのバイナリ**を使用するため、ターゲットサーバーに`cargo`やRustは必要ありません。

## デプロイ方法
まず、Cargo.tomlでアプリケーション名を確認します：
```toml
[package]
name = "myapp" # これがバイナリ名です
version = "0.1.0"
```

対象サーバーのアーキテクチャ用に本番バイナリをビルドします：

<!-- <snip id="build-command" inject_from="yaml" template="sh"> -->
```sh
cargo build --release
```
<!-- </snip>-->

そして、バイナリを`config/`フォルダと一緒にサーバーにコピーします。その後、サーバー上で`myapp start`を実行できます。

```sh
# ビルド後、バイナリは./target/release/に配置されます
./target/release/myapp start
```

以上です！

**すべての作業内容**が**単一の**バイナリに埋め込まれるよう特別な配慮をしているため、サーバー上にはそのバイナリ以外何も必要ありません。

## 本番設定の確認

本番環境へのデプロイ時に確認し、適切に設定することが重要な設定セクションがいくつかあります：

- ロガー：

<!-- <snip id="configuration-logger" inject_from="code" template="yaml"> -->
```yaml
# アプリケーションのロギング設定
logger:
  # ロギングを有効化または無効化
  enable: true
  # きれいなバックトレースを有効化（RUST_BACKTRACE=1を設定）
  pretty_backtrace: true
  # ログレベル、オプション：trace、debug、info、warn、error
  level: debug
  # ロギング形式を定義。オプション：compact、pretty、json
  format: compact
  # デフォルトでは、ロガーはあなたのコードまたは`loco`フレームワークからのログのみをフィルタリングします。すべてのサードパーティライブラリを表示するには
  # 以下の行のコメントを解除して、この設定を有効にしてロガーフィルターをオーバーライドできます。
  # override_filter: trace
```
<!-- </snip>-->
 

- サーバー：
<!-- <snip id="configuration-server" inject_from="code" template="yaml"> -->
```yaml
server:
  # サーバーがリッスンするポート。サーバーのバインディングは0.0.0.0:{PORT}
  port: {{ get_env(name="NODE_PORT", default=5150) }}
  # メーラーが参照するUIのホスト名またはIPアドレス
  host: http://localhost
```
<!-- </snip>-->


- データベース：
<!-- <snip id="configuration-database" inject_from="code" template="yaml"> -->
```yaml
database:
  # データベース接続URI
  uri: {{get_env(name="DATABASE_URL", default="postgres://loco:loco@localhost:5432/loco_app")}}
  # 有効にすると、SQLクエリがログに記録されます
  enable_logging: false
  # 接続取得時のタイムアウト期間を設定
  connect_timeout: 500
  # 接続を閉じるまでのアイドル期間を設定
  idle_timeout: 500
  # プールの最小接続数
  min_connections: 1
  # プールの最大接続数
  max_connections: 1
  # アプリケーションロード時にマイグレーションを実行
  auto_migrate: true
  # アプリケーションロード時にデータベースを切り詰めます。これは危険な操作です。開発環境またはテストモードでのみこのフラグを使用してください
  dangerously_truncate: false
  # アプリケーションロード時にスキーマを再作成します。これは危険な操作です。開発環境またはテストモードでのみこのフラグを使用してください
  dangerously_recreate: false
```
<!-- </snip>-->


- メーラー：
<!-- <snip id="configuration-mailer" inject_from="code" template="yaml"> -->
```yaml
mailer:
  # SMTPメーラー設定
  smtp:
    # SMTPメーラーを有効/無効化
    enable: true
    # SMTPサーバーホスト。例：localhost、smtp.gmail.com
    host: {{ get_env(name="MAILER_HOST", default="localhost") }}
    # SMTPサーバーポート
    port: 1025
    # セキュア接続（SSL/TLS）を使用
    secure: false
    # auth:
    #   user:
    #   password:
```
<!-- </snip>-->

- キュー：
<!-- <snip id="configuration-queue" inject_from="code" template="yaml"> -->
```yaml
queue:
  kind: Redis
  # Redis接続URI
  uri: {{ get_env(name="REDIS_URL", default="redis://127.0.0.1") }}
  # 起動時にRedisのすべてのデータを危険にフラッシュします。危険な操作です。開発環境またはテストモードでのみこのフラグを使用してください
  dangerously_flush: false
```
<!-- </snip>-->

- JWTシークレット：
<!-- <snip id="configuration-auth" inject_from="code" template="yaml"> -->
```yaml
auth:
  # JWT認証
  jwt:
    # トークン生成と検証用のシークレットキー
    secret: PqRwLF2rhHe8J22oBeHy
    # トークンの有効期限（秒単位）
    expiration: 604800 # 7日間
```
<!-- </snip>-->

## `loco doctor`の実行

サーバー上で`loco doctor`を実行して、環境の接続状態を確認できます。 

```sh
$ myapp doctor --production
```

## 生成

Locoはデプロイメントインフラストラクチャの作成を可能にするデプロイメントテンプレートを提供しています。

```sh
$ cargo loco generate deployment --help
デプロイメントインフラストラクチャを生成

使用方法: myapp-cli generate deployment [OPTIONS] <KIND>

引数:
  <KIND>  [使用可能な値: docker, shuttle, nginx]
```

<!-- <snip id="generate-deployment-command" inject_from="yaml" template="sh"> -->

```sh
cargo loco generate deployment docker

追加: "dockerfile"
追加: ".dockerignore"
* Dockerfileが正常に生成されました。
* Dockerignoreが正常に生成されました
```

<!-- </snip>-->

デプロイメントオプション：

1. Docker：

- ビルドとデプロイに対応したDockerfileを生成します。
- .dockerignoreファイルを作成します。

2. Shuttle：

- shuttleメイン関数を生成します。
- `shuttle-runtime`と`shuttle-axum`を依存関係として追加します。
- デプロイメント用のbinエントリポイントを追加します。

3. Nginx：

- リバースプロキシ用のnginx設定ファイルを生成します。

デプロイメントのニーズに最適なオプションを選択してください。楽しいデプロイを！

別のクラウドでのデプロイをご希望の場合は、プルリクエストをお気軽に開いてください。あなたの貢献を大歓迎します！
