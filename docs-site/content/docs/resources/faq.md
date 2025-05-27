+++
title = "よくある質問"
description = "よくある質問への回答。"
date = 2021-05-01T19:30:00+00:00
updated = 2021-05-01T19:30:00+00:00
draft = false
weight = 2
sort_by = "weight"
template = "docs/page.html"

[extra]
toc = true
top = false
flair =[]
+++

<details>
<summary>コードを自動でリロードするにはどうすればいいですか？</summary>

[cargo watchexec](https://crates.io/crates/watchexec)を試してみてください：

```
$ watchexec --notify -r -- cargo loco start
```

または[bacon](https://github.com/Canop/bacon)

```
$ bacon run
```

</details>
<br/>
<details>
<summary>タスクや他の機能を実行するには`cargo`が必要ですか？</summary>
`cargo`経由で実行する必要はありませんが、開発中は強く推奨されます。`--release`でビルドした場合、バイナリにはコードを含むすべてが含まれ、`cargo`やRustは不要です。
</details>

<br/>

<details>
<summary>これは本番対応ですか？</summary>

Locoはまだ初期段階ですが、そのルーツはそうではありません。これは`Hyperstackjs.io`のほぼ書き直しで、Hyperstackは本番対応の内部Rails風フレームワークに基づいています。

Locoの大部分はAxum、SeaORM、その他の安定したフレームワーク周りの接着コードなので、それを考慮できます。

現段階のバージョン0.1.xでは、問題が発生した場合は_採用して問題を報告する_ことをお勧めします。

</details>

<br/>
<details>
<summary>Locoでカスタムミドルウェアを追加する</summary>
LocoはAxumミドルウェアと互換性があります。カスタム構造体で`FromRequestParts`を実装し、コントローラー内で統合するだけです。
</details>

<br/>

<details>
<summary>Locoでカスタム状態やレイヤーを注入する？</summary>
はい、`Hooks::after_routes`を実装することで可能です。このフックはLocoが既に構築したAxumルーターを受け取り、ニーズに合う利用可能なAxum関数をシームレスに追加できます。

ルートや（404）フォールバックハンドラーがLocoのミドルウェアの影響を受ける必要がある場合は、ミドルウェアがインストールされる前に呼び出される`Hooks::before_routes`に追加できます。
</details>

<br/>
