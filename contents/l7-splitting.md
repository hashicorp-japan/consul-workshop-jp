# L7 Traffic Managementを試す

次に`service-splliting`の機能を試してみます。この手順を完了させるためには`l7-routing`の手順を完了させておく必要があります。

すでに手順が完了している方は`Ctrl+C`でkillしてから以下のコマンドで両方のプロセスを一旦停止してください。

Service Splittingは各サブセットや異なるサービスに対して重み付けルーティングを行うための機能です。これを利用することでバージョンアップや、アプリケーションをリライトして別のサービスとして一度立ち上げるなどの際に安全にトラフィックの切り替えを行うことができます。

```shell
$ docker-compose down
$ docker-compose -f docker-compose-v2.yaml down
```

## 設定を変更する

既存のルーティングの設定は一旦解除します。以下のコマンドで設定を上書きしてください。

```shell
$ cat << EOF > central/api_service_router.hcl
kind = "service-router"
name = "greetings-api"
routes = [
  {
    match {
      http {
        header = [
          {
            name  = "canary"
            exact = "*"
          },
        ]
      }
    }

    destination {
      service = "greetings-api"
    }

  },
]
EOF

$ $ cat << EOF > central/api_service_resolver.hcl
kind = "service-resolver"
name = "greetings-api"

subsets = {
  v1 = {
    filter = "Service.Meta.version == 1"
  }
  v2 = {
    filter = "Service.Meta.version == 2"
  }
}
EOF
```

変更点は`canary`にどのバリューがセットされていても、`greeting-api`にルーティングされるように設定し、サブセットの指定を削除しています。`api_service_resolver`ではデフォルトのサブセットを削除しています。これによりラウンドロビンで各サブセットにリクエストが振られることになります。


一度起動して動作を確かめてみましょう。別々のターミナルで以下を実行します。

```shell
$ docker-compose up
$ docker-compose -f docker-compose-v2.yaml up
```

起動が完了したら動作を確認してみます。

```console
$ curl  127.0.0.1:9090/ --header "canary: 1"
Greetings From API -> <200,hi i am v1

$ curl  127.0.0.1:9090/ --header "canary: 1"
Greetings From API -> <200,hi i am v2
```

`v1`,`v2`にリクエストが振られています。

## Splittingの設定を作成する

次にSplittingの設定を作ります。ここでは100:0, 50:50, 0:100の割合で各サブセットにリクエストを分散する設定を行なっています。

```shell
$ cat << EOF > 100-0.json
{
  "kind": "service-splitter",
  "name": "greetings-api",
  "splits": [
    {
      "weight": 100,
      "service_subset": "v1"
    },
    {
      "weight": 0,
      "service_subset": "v2"
    }
  ]
}
EOF

$ cat << EOF > 50-50.json
{
  "kind": "service-splitter",
  "name": "greetings-api",
  "splits": [
    {
      "weight": 50,
      "service_subset": "v1"
    },
    {
      "weight": 50,
      "service_subset": "v2"
    }
  ]
}
EOF

$ cat << EOF > 0-100.json
{
  "kind": "service-splitter",
  "name": "greetings-api",
  "splits": [
    {
      "weight": 0,
      "service_subset": "v1"
    },
    {
      "weight": 100,
      "service_subset": "v2"
    }
  ]
}
EOF
```

これをConsulに順番に反映させてみましょう。

```shell
$ curl localhost:8500/v1/config -XPUT -d @100-0.json
```

これで100:0でv1にリクエストが振られるようになるはずです。

```console
$ curl  127.0.0.1:9090/ --header "canary: 1"
Greetings From API -> <200,hi i am v1
```

何度か実行して動作を確認してください。

次に50:50にします。

```shell
$ curl localhost:8500/v1/config -XPUT -d @50-50.json
```

リクエストが均等に振られているでしょう。

```console
$ curl  127.0.0.1:9090/ --header "canary: 1"
Greetings From API -> <200,hi i am v1

$ curl  127.0.0.1:9090/ --header "canary: 1"
Greetings From API -> <200,hi i am v2
```

最後に0:100でv2にのみリクエストが行くようにします。

```shell
$ curl localhost:8500/v1/config -XPUT -d @0-100.json
```

確認します。何度か実行してみてください。

```console
$ curl  127.0.0.1:9090/ --header "canary: 1"
Greetings From API -> <200,hi i am v2
```

以上のように`service-splitting`の機能を使ってアプリケーションのロールアウトを実施してみました。今回はサブセットを利用しましたが、コードベースやリライトなどを実施した後の異なるサービス同士でもSplittingを行うことも可能です。

## 参考リンク
* [Service Splitting](https://www.consul.io/docs/agent/config-entries/service-splitter.html)
