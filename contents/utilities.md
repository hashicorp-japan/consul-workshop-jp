# 運用系機能色々

ここでは Consul の持つ多くの機能の一部を試してみます。
これらの機能は非常に限定的な機能ですが、使い方によってはとてもパワフルなものです。また非常に汎用的に活用できます。

主に以下の３つの機能を試します。

1. Watch (Blocking query)
1. Exec (リモート実行）
1. Lock (セマフォ)


## Watch

Watch 機能は、Consul クラスタの状態や KV など様々な情報の更新などをモニターする機能です。それらの情報がアップデートされる度に事前に設定しておいたコマンドやスクリプトなどを起動します。
現時点の Consul が Watch できる情報は以下になります。詳細は[こちら](https://www.consul.io/docs/agent/watches.html)を参照ください。

* KV pair
	* KV pair は指定した KV の値の変更をモニタリングします。
* Keyprefix
	* 指定した KV の prefix に一致した値の変更をモニタリングします。
* Event
	* ユーザー指定の Event などをモニタリングします。
* Service
	* 特定の Service の tag などの変更をモニタリングします。
* Services
	* 登録されている Service の変更をモニタリングします。
* Nodes
	* Consul クラスタの Node の変更をモニタリングします。
* Checks
	* 特定の Service のヘルスチェックなどの変更をモニタリングします。


ここでは**KV pair**と**Event**の二つを試してみます。

### KV pair

特定の KV の値の更新により発動する Watch を作成します。まず、以下のコマンドで KV を作成してください。

```shell
$ consul kv put workshop/kv_watch foo
```

これで Consul の KV 上の`workshop/kv_watch`というキーに`foo`という値が入りました。次にこの値の変更をモニタリングする Watch を作成します。

```shell
$ consul watch -type=key -key=workshop/kv_watch "jq -r .Value | base64 -D && echo"
```

このコマンドにより`workshop/kv_watch`の値に変更があった際に、任意のコマンド（最後のパラメーター）を実行します。この場合は、更新された値を読み取り、Base64 のデコードを行い表示します。最後の`echo`コマンドは改行するためだけのものです。
ちなみに Consul は KV へは Base64 でエンコードされた値を保持します。これはバイナリデータなども保管できるようにしているためです。


実行すると、まず現在の値を表示し、Long running のプロセスとして実行され続けます。この状態で別のターミナルを開き、KV の値に変更を加えてみます。

```console
$ consul kv put workshop/kv_watch bar
Success! Data written to: workshop/kv_watch
```

Consul Watch が起動しているターミナルを見ると以下のようになっているはずです。

```console
$ consul watch -type=key -key=workshop/kv_watch "jq -r .Value | base64 -D && echo"
foo
bar
```

KV の値を何回か更新してみて、その度に Watch がモニタリングしていることを確認してください。

この機能により、例えば KV の値をアップデートして、それでシステムやアプリの再設定を行う、といった事ができます。

### Event

Event がユーザーが任意の Event を Watch へ送る事ができる機能です。

以下のコマンドを打って Event をモニタリングする Watch を作成してください。

```shell
$ consul watch -type=event -name=broadcast "jq -r .[].Payload | base64 -D && echo"
```

ここでは、`broadcast`という名前の Event をモニタリングし、その Event が届く度に任意のコマンドを実行する Watch を作成しています。

それではまた別のターミナルから Event を送ってみましょう。

```shell
$ consul event -name=broadcast "Sample Payload"
```

Watch のターミナルに書き込んだ Payload が表示されていると思います。Event を何度か送って試してみてください。

以上が、Watch の機能の紹介でした。

## Exec (リモート実行)

Exec 機能はクラスタ内のサーバーおよびクライアントで、コマンドをリモート実行する機能です。コマンドの詳細については[こちら](https://www.consul.io/docs/commands/exec.html)。

コマンドのオプションにより、実行対象のノードはフィルタリングできます。

以下のコマンドを叩いてください。

```shell
consul exec "uptime"
```

オプションでフィルタリングしない場合、全てのクラスタメンバでコマンド実行します。
ここでは、全てのメンバに対して`uptime`コマンドを実行させました。フィルタリングの例としては、

```shell
consul exec -node "^client" "uptime"  # 'client'の文字列から始まる名前のノードだけで実行

consul exec -service "web" "uptime"   # 'web'というServiceが登録してあるノードだけで実行
```

などがあります。
Consul の KV と組み合わせて、ノードでの実行結果を保存することも可能です。

```shell
consul exec 'consul kv put machines/$(hostname)/uptime "$(uptime)"'
```

ここでは、各ノードに`uptime`を実行させ、その結果を KV に書き込むことで、全てのノードの Uptime を一度に取得することができます。

結果は以下のようになります。

```console
$consul kv get machines/client001/uptime
 15:04:40 up  8:42,  0 users,  load average: 0.00, 0.02, 0.00
$consul kv get machines/client002/uptime
 15:04:40 up  8:41,  0 users,  load average: 0.07, 0.03, 0.01
```

以上が、Exec の機能の紹介でした。

## Lock (セマフォ)

Lock 機能は、シンプルな排他処理を行いたい際に便利です。
デフォルトでは一つのロックが作成され、ただ一つのノードだけが選出されます。ロックの数は複数個の指定も可能です。カウンティングセマフォ）。コマンドの詳細は[こちら](https://www.consul.io/docs/commands/lock.html)

まずはローカルで動いている Consul Agent で試してみましょう。以下のスクリプトを実行してみてください。


```shell
#!/bin/bash

for i in {1..5}
do
  consul lock workshop/lock "echo Process $i has the lock && sleep 3 && echo Process $i has released the lock" &
done

wait
```

ここでは Consul KV 上に`workshop/lock`というロックを作成しています。そして、バックグラウンドプロセスを５つ作成し、ロックを取得できたプロセスから順次コマンドを実行しています。各プロセスは自らのプロセス番号を表示し、３秒間スリープした後でプロセス終了とともにロックを解放します。ロックが解放されるとともに、別のプロセスがロックを取得し、また同様な処理を行います。以下のような出力になります。

```console
Process 2 has the lock
Process 2 has released the lock
Process 1 has the lock
Process 1 has released the lock
Process 5 has the lock
Process 5 has released the lock
Process 3 has the lock
Process 3 has released the lock
Process 4 has the lock
Process 4 has released the lock
```

ロックを取得できたプロセス順に処理が行われたことがわかります。
ロックは複数個準備することもできます。先ほどのスクリプトに`-n=2`というオプションでロックを二つ準備するようにしてみます。

```shell
#!/bin/bash

for i in {1..5}
do
  consul lock -n=2 workshop/lock "echo Process $i has the lock && sleep 3 && echo Process $i has released the lock" &
done

wait
```

実行すると以下のようになります。

```console
Process 2 has the lock
Process 5 has the lock
Process 2 has released the lock
Process 5 has released the lock
Process 3 has the lock
Process 1 has the lock
Process 3 has released the lock
Process 4 has the lock
Process 1 has released the lock
Process 4 has released the lock
```

二つのプロセスが同時に起動するようなったことがわかります。この仕組みを使うことで、例えば常に一定数のサービスやコマンドが起動されるよう制御できます。

クラスタ上のノードで同じロックを用いてサービスの起動などを行うことで、簡単に可用性を持つサービスを作成できます。
例：常に５つのサービスを起動していたい場合、ロックを 5 つ準備し（`-n=5`)、クラスタ上では 6 つのサービスを consul lock で起動する。どれか一つのサービスが終了もしくはロックを解放すると、残りの一つが起動する。


## まとめ

ここでは、Watch, Exec, Lock といった機能を試しました。これらは Consul のメインの機能ではありませんが、クラスタ上でのオペレーションを助けてくれる便利な機能です。また、これらの機能を組み合わせることで、非常に多岐に渡るユースケースに対応できます。


