+++
title = "Mailers"
description = ""
date = 2021-05-01T18:10:00+00:00
updated = 2021-05-01T18:10:00+00:00
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

メーラーは既存の`loco`バックグラウンドワーカーインフラストラクチャを使用して、バックグラウンドでメールを配信します。すべてがシームレスに動作します。

# メールの送信

既存のメーラーを使用するには、主にコントローラーで：

```rust
use crate::{
    mailers::auth::AuthMailer,
}

// controllers/auth.rsで
async fn register(
    State(ctx): State<AppContext>,
    Json(params): Json<RegisterParams>,
) -> Result<Response> {
    // .. ユーザーを登録 ..
    AuthMailer::send_welcome(&ctx, &user.email).await.unwrap();
}
```

これによりメール配信ジョブがキューに追加されます。配信は後でバックグラウンドで実行されるため、アクションは即座に完了します。

## 設定

メーラーの設定は`config/[stage].yaml`ファイルで行います。デフォルトの設定は以下の通りです：

```yaml
# メーラー設定
mailer:
  # SMTPメーラー設定
  smtp:
    # SMTPメーラーを有効/無効化
    enable: true
    # SMTPサーバーホスト。例：localhost、smtp.gmail.com
    host: {{/* get_env(name="MAILER_HOST", default="localhost") */}}
    # SMTPサーバーポート
    port: 1025
    # セキュア接続（SSL/TLS）を使用
    secure: false
    # auth:
    #   user:
    #   password:
```

メーラーはSMTPサーバーにメールを送信することで動作します。SendGrid（SMTPリレーオプションを選択）を使用する設定例：

```yaml
# メーラー設定
mailer:
  # SMTPメーラー設定
  smtp:
    # SMTPメーラーを有効/無効化
    enable: true
    # SMTPサーバーホスト。例：localhost、smtp.gmail.com
    host: {{/* get_env(name="MAILER_HOST", default="smtp.sendgrid.net") */}}
    # SMTPサーバーポート
    port: 587
    # セキュア接続（SSL/TLS）を使用
    secure: true
    auth:
      user: "apikey"
      password: "your-sendgrid-api-key"
```

### デフォルトのメールアドレス

すべてのメール送信タスクでメールアドレスを指定する以外に、メーラーごとにデフォルトのメールアドレスをオーバーライドできます。

まず、`Mailer`トレイトの`opts`関数をオーバーライドします。この例では`AuthMailer`用です：

```rust
impl Mailer for AuthMailer {
    fn opts() -> MailerOpts {
        MailerOpts {
            from: // 送信元メールを設定,
            ..Default::default()
        }
    }
}
```

### 開発環境でのメールキャッチャーの使用

`MailHog`や`mailtutan`（Rustで書かれています）のようなアプリを使用できます：

```
$ cargo install mailtutan
$ mailtutan
listening on smtp://0.0.0.0:1025
listening on http://0.0.0.0:1080
```

これにより、ローカルのSMTPサーバーと、メールを受信時に「キャッチ」して表示する優れたUIが`http://localhost:1080`で起動します。

そして、`development.yaml`に以下を追加します：

```yaml
# メーラー設定
mailer:
  # SMTPメーラー設定
  smtp:
    # SMTPメーラーを有効/無効化
    enable: true
    # SMTPサーバーホスト。例：localhost、smtp.gmail.com
    host: localhost
    # SMTPサーバーポート
    port: 1025
    # セキュア接続（SSL/TLS）を使用
    secure: false
```

これで、メーラーワーカーは`localhost`のSMTPサーバーにメールを送信するようになります。

## メーラーの追加

メーラーを生成できます：

```sh
cargo loco generate mailer <mailer name>
```

または、動作の仕組みを理解したい場合は手動で定義することもできます。`mailers/auth.rs`に以下を追加します：

```rust
static welcome: Dir<'_> = include_dir!("src/mailers/auth/welcome");
impl AuthMailer {
    /// 指定されたユーザーにウェルカムメールを送信
    ///
    /// # エラー
    ///
    /// メール送信が失敗した場合
    pub async fn send_welcome(ctx: &AppContext, _user_id: &str) -> Result<()> {
        Self::mail_template(
            ctx,
            &welcome,
            Args {
                to: "foo@example.com".to_string(),
                locals: json!({
                  "name": "joe"
                }),
                ..Default::default()
            },
        )
        .await?;
        Ok(())
    }
}
```

各メーラーには決まったフォルダ構造があります：

```
src/
  mailers/
    auth/
      welcome/      <-- メールのすべての部分、すべてのテンプレート
        subject.t
        html.t
        text.t
    auth.rs         <-- メーラー定義
```

### メーラーの実行
メーラーはバックグラウンドワーカーとして動作するため、ジョブを処理するには別途ワーカーを実行する必要があります。デフォルトの起動コマンド`cargo loco start`はワーカーを起動しないため、別途実行する必要があります：

ワーカーを実行するには、以下のコマンドを使用します：
```bash
cargo loco start --worker
```

サーバーとワーカーを同時に実行するには、以下のコマンドを使用します：
```bash
cargo loco start --server-and-worker
```

# テスト

ワークフローの一部として送信されるメールのテストは複雑なタスクになる可能性があり、ユーザー登録時のメール認証やユーザーパスワードメールの確認など、様々なシナリオの検証が必要です。主な目標は、ワークフローで送信されたメールの数を調べ、メールの内容を確認し、データのスナップショットを可能にすることで、テストプロセスを効率化することです。

`Loco`では、スタブテストメール機能を導入しました。基本的に、メールは実際には送信されず、代わりにテストコンテキストの一部として、メールの数とその内容に関する情報を収集します。

## 設定

テストでスタブを有効にするには、YAMLファイルのメーラーセクションの設定に以下のフィールドを追加します：

```yaml
mailer:
  stub: true
```

注意：メール送信者が[ワーカー](@/docs/processing/workers.md)プロセス内で動作する場合は、ワーカーモードがForegroundBlockingに設定されていることを確認してください。

スタブを設定したら、ユニットテストに進み、以下の例に従ってください：

## テストの作成

テストの説明：

- コードの一部としてメール送信を担当するエンドポイントにHTTPリクエストを作成します。
- コンテキストからメーラーインスタンスを取得し、送信されたメールの数とその内容に関する情報を含むdeliveries()関数を呼び出します。

```rust
use loco_rs::testing::prelude::*;

#[tokio::test]
#[serial]
async fn can_register() {
    configure_insta!();

    request::<App, Migrator, _, _>(|request, ctx| async move {
        // Create a request for user registration.

        // Now you can call the context mailer and use the deliveries function.
        with_settings!({
            filters => cleanup_email()
        }, {
            assert_debug_snapshot!(ctx.mailer.unwrap().deliveries());
        });
    })
    .await;
}
```

