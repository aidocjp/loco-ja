+++
title = "スターター"
date = 2021-12-19T08:00:00+00:00
updated = 2021-12-19T08:00:00+00:00
draft = false
weight = 4
sort_by = "weight"
template = "docs/page.html"

[extra]
toc = true
top = false
flair =[]
+++

開発体験をよりスムーズにするために設計されたLocoの事前定義されたボイラープレートで、プロジェクトのセットアップを簡素化します。開始するには、CLIをインストールして、ニーズに合ったテンプレートを選択してください。

<!-- <snip id="quick-installation-command" inject_from="yaml" template="sh"> -->
```sh
cargo install loco
cargo install sea-orm-cli # DBが必要な場合のみ
```
<!-- </snip> -->

スターターを作成：

<!-- <snip id="loco-cli-new-from-template" inject_from="yaml" template="sh"> -->
```sh
❯ loco new
✔ ❯ App name? · myapp
✔ ❯ What would you like to build? · Saas App with client side rendering
✔ ❯ Select a DB Provider · Sqlite
✔ ❯ Select your background worker type · Async (in-process tokio async tasks)

🚂 Loco app generated successfully in:
myapp/

- assets: You've selected `clientside` for your asset serving configuration.

Next step, build your frontend:
  $ cd frontend/
  $ npm install && npm run build
```
<!-- </snip> -->

## 利用可能なスターター

### SaaSスターター

SaaSスターターは、UIとREST APIの両方を必要とするプロジェクト向けの全て込みのセットアップです。UIについては、このスターターはクライアントサイドアプリまたは従来のサーバーサイドテンプレート（またはその組み合わせ）をサポートします。

**UI**

- ReactとViteで構築されたフロントエンドスターター（お好みのフレームワークに簡単に置き換え可能）。
- フロントエンドビルドを指してフォールバックインデックスを含む静的ミドルウェア。またはサーバーサイドテンプレート用の静的アセット用に設定することもできます。
- i18n設定を含む、サーバーサイドテンプレート用に設定されたTeraビューエンジン。テンプレートとi18nアセットは`assets/`にあります。

**Rest API**

- サービスの健全性をチェックする`ping`と`health`エンドポイント。すべてのエンドポイントを確認するには`cargo loco routes`コマンドを使用
- ユーザーテーブルと認証ミドルウェア。
- 認証ロジックとユーザー登録を持つユーザーモデル。
- パスワード忘れAPIフロー。
- ウェルカムメールを送信し、パスワード忘れリクエストを処理するメーラー。

#### サーバーサイドテンプレート用のアセット設定

SaaSスターターはフロントエンドクライアントサイドアセット用に事前設定されています。画像やスタイルなどのアセットを含むサーバーサイドテンプレートレンダリングを使用したい場合は、そのためのアセットミドルウェアを設定できます：

`config/development.yaml`で、サーバーサイド設定のコメントを外し、クライアントサイド設定をコメントアウトしてください。

```yaml
    # server-side static assets config
    # for use with the view_engine in initializers/view_engine.rs
    #
    static:
      enable: true
      must_exist: true
      precompressed: false
      folder:
        uri: "/static"
        path: "assets/static"
      fallback: "assets/static/404.html"
    fallback:
      enable: false
    # client side app static config
    # static:
    #   enable: true
    #   must_exist: true
    #   precompressed: false
    #   folder:
    #     uri: "/"
    #     path: "frontend/dist"
    #   fallback: "frontend/dist/index.html"
    # fallback:
    #   enable: false
```


### Rest APIスターター

フロントエンドなしでREST APIのみが必要な場合は、Rest APIスターターを選択してください。後でフロントエンドを提供することに決めた場合は、`static`ミドルウェアを有効にして、設定を`frontend`配布フォルダーに向けるだけです。

### 軽量サービススターター

コントローラーとビュー（レスポンススキーマ）に焦点を当てた軽量サービススターターはミニマリスティックです。データベース、フロントエンド、ワーカー、またはLocoが提供するその他の機能なしでREST APIサービスが必要な場合、これが理想的な選択です！
