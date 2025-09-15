# Consul Service Configuration

Consul には Key Value のデータストアが内包されており、これを利用することで様々な処理を行うことができます。

* 各マイクロサービスへのコンフィグレーションの配布(Centerized Configurations)
* Consul 管理は以下のサービスのコンフィグレーションの自動アップデート(Consul Template)
* KVS の値を監視して更新などをトリガーにしたイベント処理(Watch)

などです。

Watch は別の章で扱いますのでここでは`Centerized Configurations`を試してみたいと思います。


## Consul KV の使い方

まず、Consul KV を使って簡単なデータの put/get をしてみましょう。

```console 
$ consul agent -dev
$ consul kv put my-first-kv/consulis useful
$ consul kv get my-first-kv/consulis
useful
```

`consulis`というキーに`useful`という値を入れて取り出しました。とても簡単です。Consul 内のデータは Consul クラスタ内のノード同士でレプリケーションし、ユーザは意識することなくデータの冗長性を保つことができます。

余裕のある方は Web ブラウザの`Key/Value`のタブからも確認ができるのでそこから値を確認したり編集して、再度`consul kv get`をしてみて下さい。

`Ctrl+C`で Consul を停止してください。

## Centerized Configurations

Workshop 用のディレクトリがない場合は任意のディレクトリを作りましょう。

```shell
$ mkdir consul-workshop
$ cd consul-workshop
```

ここでは簡単な Web アプリを使って Consul から Web アプリに設定を注入してみます。Consul の KVS に設定を格納し、それを各アプリケーションが取得します。まずはアプリのレポジトリをクローンして、セットアップをします。

```shell
$ git clone https://github.com/tkaburagi/consul-config-spring
$ cd consul-config-spring
$ mkdir consul_data
$ DIR=$(pwd)
$ cat << EOF > consul.d/consul-config.hcl
data_dir  = "${DIR}/consul_data/consul-config-demo-data"
datacenter = "dc1"

server = true

bootstrap_expect = 1
ui               = true

bind_addr   = "0.0.0.0"
client_addr = "0.0.0.0"

ports {
  grpc = 8502
}

connect {
  enabled = true
}

enable_central_service_config = true
EOF
```

アプリケーションをビルドして Docker コンテナとして起動します。Consul サーバがすでに起動している場合は　`Ctrl+C`で停止してください。

**sudo が必要な方は`run-sudo.sh`を実行してください。**

```shell
$ chmod +x run.sh
$ ./run.sh
```

`http://127.0.0.1:8500`にブラウザでアクセスし、起動していることを確認してください。

7 つのアプリケーションインスタンスが起動しているはずです。

それぞれがコンテナで起動し、1010 - 7070 のポートでローカルホストからアクセスできるはずです。

```console
$ curl 127.0.0.1:1010
hi from APP_1 at 172.23.0.4
```

ここでは`hi`の文字列は`applications.properties`でセットされているコンフィグから読み取っています。

例えば、全アプリケーションのこの設定を変更し、出力するメッセージを変更する際、通常は設定を変更し、アプリケーションをビルドし直して、再デプロイするといった運用が必要です。

これは単なるメッセージの文字列の設定だけでなく、環境ごとに変わるような DB の設定、セキュリティやログレベルといった全ての設定で同じことが言えます。

Consul でそのような複数のアプリに対する設定変更をどのように実現できるかをこのあと試してみます。

**別ターミナルを立ち上げて**アプリからのレスポンス内容を監視しておきます。

```console
$ ./watch.sh
hi from APP_1 at 172.23.0.4
hi from APP_2 at 172.23.0.6
hi from APP_3 at 172.23.0.3
hi from APP_4 at 172.23.0.8
hi from APP_5 at 172.23.0.7
hi from APP_6 at 172.23.0.2
hi from APP_7 at 172.23.0.5
```

## Consul にコンフィグレーションを書き込む

このアプリでは下記の`bootstrap.yml`の設定により、`config`という prefix のディレクトリの中に保存されている`data`という設定をアプリのコンフィグレーションとして扱うように設定しています。

これは Spring Cloud Consul の機能を利用して抽象的な設定が可能ですが、他のアプリでも同じように実装できるはずです。

```yaml
spring:
  cloud:
    consul:
      host: 10.5.0.2
      port: 8500
      config:
        format: YAML
        enabled: true
        prefix: config
        data: data
```

Consul の KVS を利用して、設定内容を書き込みます。

```shell
$ consul kv put config/application/data @app_config.yml
```

`app_config.yml`にはメトリクスやログレベルの設定の他に、`app.message`という、先ほどの`hi`とセットされている内容を`HEY HEY HEY`にオーバーライドしています。

```console
$ consul kv get config/application/data
#config/application/data
management:
  endpoints:
    web:
      exposure:
        include: "*"
logging:
  level:
    org.springframework.web: DEBUG

app:
  message: "HEY HEY HEY"
```

あとはアプリを再起動するだけです。

> sudo で Docker を起動している場合は`visudo`でパスワードなしで Root ユーザで起動できるようにして下さい。

> sudo で Docker を起動している方はこちらを実行して下さい。
> `mv updateconfig.sudo.sh updateconfig.sh`

```console
$ docker ps
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS                                                                      NAMES
076eb24c6755        consulconfigspring_app_5   "java -jar /app.jar"     13 minutes ago      Up 2 minutes        0.0.0.0:5050->9090/tcp                                                     consulconfigspring_app_5_1
3ad04c67434f        consulconfigspring_app_7   "java -jar /app.jar"     13 minutes ago      Up 2 minutes        0.0.0.0:7070->9090/tcp                                                     consulconfigspring_app_7_1
7ad51af17aa2        consulconfigspring_app_3   "java -jar /app.jar"     13 minutes ago      Up 2 minutes        0.0.0.0:3030->9090/tcp                                                     consulconfigspring_app_3_1
d150cff0cb63        consulconfigspring_app_4   "java -jar /app.jar"     13 minutes ago      Up 2 minutes        0.0.0.0:4040->9090/tcp                                                     consulconfigspring_app_4_1
a6374bd453f6        consulconfigspring_app_6   "java -jar /app.jar"     13 minutes ago      Up 2 minutes        0.0.0.0:6060->9090/tcp                                                     consulconfigspring_app_6_1
63830c767cce        consulconfigspring_app_2   "java -jar /app.jar"     13 minutes ago      Up About a minute   0.0.0.0:2020->9090/tcp                                                     consulconfigspring_app_2_1
55bfd848f9c2        consulconfigspring_app_1   "java -jar /app.jar"     13 minutes ago      Up About a minute   0.0.0.0:1010->9090/tcp                                                     consulconfigspring_app_1_1
71f344799e6d        consul:1.6.0               "docker-entrypoint.s…"   15 minutes ago      Up 13 minutes       8300-8302/tcp, 8301-8302/udp, 8600/tcp, 8600/udp, 0.0.0.0:8500->8500/tcp   consulconfigspring_consul_1
```

この中から Spring アプリの IMAGE 名の*prefix*を取得して下さい。この場合だと`consulconfigspring_app`です

```
updateconfig.sh内の

"IMAGE_NAME="を

"IMAGE_NAME=consulconfigspring_app"

に変更して下さい。
```

```shell
$ ./updateconfig.sh
```

これでアプリが順番で再起動します。`watch.sh`の出力結果を見ながら、アプリの設定が全部のインスタンスに反映されていることを確認しましょう。

```
HEY HEY HEY from APP_1 at 172.23.0.4
HEY HEY HEY from APP_2 at 172.23.0.6
HEY HEY HEY from APP_3 at 172.23.0.3
HEY HEY HEY from APP_4 at 172.23.0.8
HEY HEY HEY from APP_5 at 172.23.0.7
HEY HEY HEY from APP_6 at 172.23.0.2
HEY HEY HEY from APP_7 at 172.23.0.1
```

このようにアプリに一切変更を加えることなく設定を Consul から取得し反映させることができます。

## Watch の利用

次に Consul の`watch`機能を利用し、KVS の更新をトリガーにスクリプトを実行させ、自動でアプリケーションの起動を行います。

`${CONFIG_DIR}/consul-config.hcl`を変更します。

```shell
$ cd consul-workshop/consul-config-spring
$ DIR=$(pwd)
$ cat << EOF > consul.d/consul-config.hcl
data_dir  = "/data/localdata"
datacenter = "dc1"

server = true

bootstrap_expect = 1
ui               = true

bind_addr   = "0.0.0.0"
client_addr = "0.0.0.0"

ports {
  grpc = 8502
}

connect {
  enabled = true
}

enable_central_service_config = true

"watches" = {
  "args" = ["${DIR}/updateconfig.sh"]

  "handler_type" = "script"

  "key" = "config/application/data"

  "type" = "key"
}
EOF
```

`config/application/data`の更新をキーに、`updateconfig.sh`を invoke しています。
以下のコマンドで Consul の設定を反映させましょう。

```shell
$ consul reload
```

再度アプリの設定を変更します。次は Web GUI から実行してみましょう。`http://127.0.0.1:8500/ui/dc1/kv/config/application/data/edit`こちらにアクセスし、`HEY HEY HEY`の値を任意の文字列に変更してください。

変更後`Save`をクリックすると、`updateconfig.sh`の処理が開始されるはずです。

`watch.sh`の出力ターミナルを見て、変更が自動で反映されていることを確認してみましょう。

```
How is Consul? from APP_1 at 172.23.0.4
How is Consul? from APP_2 at 172.23.0.6
How is Consul? from APP_3 at 172.23.0.3
How is Consul? from APP_4 at 172.23.0.8
How is Consul? from APP_5 at 172.23.0.7
How is Consul? from APP_6 at 172.23.0.2
How is Consul? from APP_7 at 172.23.0.9
```
Consul を利用することで設定をアプリからは透過的に変更することができることがわかりました。マイクロサービスなどを採用してアプリケーションの数が増えてきたときこの機能により数十、数百のアプリに対して自動で設定を反映させることが可能です。

最後に`Ctr+C`で抜けて全コンテナを停止しておきましょう。

```shell
$ docker-compose down
```

## 参考リンク
* [KV](https://www.consul.io/docs/agent/kv.html)
* [Consul Connect Configuration](https://www.consul.io/docs/connect/configuration.html)
