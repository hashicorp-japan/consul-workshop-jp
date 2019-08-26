# 運用系機能色々

ここではConsulの持つ多くの機能の一部を試してみます。
これらの機能は非常に限定的な機能ですが、使い方によってはとてもパワフルなものです。また非常に汎用的に活用できます。

主に以下の３つの機能を試します。

1. Watch (Blocking query)
1. Exec (リモート実行）
1. Lock (セマフォ)

## Watch

Watch機能は、Consulクラスタの状態やKVなど様々な情報の更新などをモニターする機能です。それらの情報がアップデートされる度に事前に設定しておいたコマンドやスクリプトなどを起動します。
現時点のConsulがWatchできる情報は以下になります。詳細は[こちら](https://www.consul.io/docs/agent/watches.html)を参照ください。

* KV pair
	* KV pairは指定したKVの値の変更をモニタリングします。
* Keyprefix
	* 指定したKVのprefixに一致した値の変更をモニタリングします。
* Event
	* ユーザー指定のEventなどをモニタリングします。
* Service
	* 特定のServiceのtagなどの変更をモニタリングします。
* Services
	* 登録されているServiceの変更をモニタリングします。
* Nodes
	* ConsulクラスタのNodeの変更をモニタリングします。
* Checks
	* 特定のServiceのヘルスチェックなどの変更をモニタリングします。


ここでは**KV pair**と**Event**の二つを試してみます。

### KV pair

特定のKVの値の更新により発動するWatchを作成します。まず、以下のコマンドでKVを作成してください。

```shell
$consul kv put workshop/kv_watch foo
```

これでConsulのKV上の`workshop/kv_watch`というキーに`foo`という値が入りました。次にこの値の変更をモニタリングするWatchを作成します。

```shell
$consul watch -type=key -key=workshop/kv_watch "jq -r .Value | base64 -D && echo"
```

このコマンドにより`workshop/kv_watch`の値に変更があった際に、任意のコマンド（最後のパラメーター）を実行します。この場合は、更新された値を読み取り、Base64のデコードを行い表示します。最後の`echo`コマンドは改行するためだけのものです。
ちなみにConsulはKVへはBase64でエンコードされた値を保持します。これはバイナリデータなども保管できるようにしているためです。


実行すると、まず現在の値を表示し、Long runningのプロセスとして実行され続けます。この状態で別のターミナルを開き、KVの値に変更を加えてみます。

```console
$consul kv put workshop/kv_watch bar
Success! Data written to: workshop/kv_watch
```

Consul Watchが起動しているターミナルを見ると以下のようになっているはずです。

```console
$consul watch -type=key -key=workshop/kv_watch "jq -r .Value | base64 -D && echo"
foo
bar
```

KVの値を何回か更新してみて、その度にWatchがモニタリングしていることを確認してください。

この機能により、例えばKVの値をアップデートして、それでシステムやアプリの再設定を行う、といった事ができます。

### Event

Eventがユーザーが任意のEventをWatchへ送る事ができる機能です。

以下のコマンドを打ってEventをモニタリングするWatchを作成してください。

```shell
$consul watch -type=event -name=broadcast "jq -r .[].Payload | base64 -D && echo"
```

ここでは、`broadcast`という名前のEventをモニタリングし、そのEventが届く度に任意のコマンドを実行するWatchを作成しています。

それではまた別のターミナルからEventを送ってみましょう。

```shell
$consul event -name=broadcast "Sample Payload"
```

Watchのターミナルに書き込んだPayloadが表示されていると思います。Eventを何度か送って試してみてください。


## Exec (リモート実行)

## Lock (セマフォ)





