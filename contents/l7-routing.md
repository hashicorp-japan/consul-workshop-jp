# L7 Traffic Management を試す

より高度な Service Mesh を実現するため Consul1.6 から L7 の Traffic Management の機能が追加されました。HTTP パスベースのルーティング、トラフィックの重み付け、フェイルオーバーなどの機能が追加され、A/B Test, Canary Upgrade, Blue/Green Deployment などが実現できます。

ここではそれらの機能を簡単なサンプルで実行してみます。

Consul の L7 Traffic Management の機能は

* Routing
* Splitting
* Resolution

の三つのタイプに分けられています。

## Service Routing

まずは`Service Routing`を扱ってみましょう。Service Routing では HTTP パスやヘッダーなどの情報を基にルーティングを制御することができ、AB テストた Canary デプロイを柔軟に行うことができます。

まずは利用するアプリケーションをクローンします。
```shell
$ cd path/to/consul-workshop
$ https://github.com/tkaburagi/consul-l7-routing
```

`docker-compose.yaml`を見ると Consul, web, api-v1 とそれぞれの Sidecar Proxy として動作する Envoy が稼働することがわかります。

## 設定を作成する

また、`central`と`consul.d`という二つのからのディレクトリが存在し、それを設定として起動時に読み込むようになっています。まずはそれぞれのサービスの設定を作っていきます。`consul.d`のディレクトリに作っていきます。

```shell
$ cat << EOF > consul.d/sidecar-for-greetings-api-v1.json
{
  "service": {
    "name": "greetings-api",
    "id": "greetings-api-v1",
    "tags": ["v1"],
    "meta": {
      "version": "1"
    },
    "address": "10.5.0.4",
    "port": 8080,
    "connect": {
      "sidecar_service": {
        "port": 19002,
        "check": {
          "name": "Connect Envoy Sidecar",
          "tcp": "10.5.0.4:19002",
          "interval": "10s"
        },
        "proxy": {
        }
      }
    }
  }
}
EOF

$ cat << EOF > consul.d/sidecar-for-greetings-api-v2.json
{
  "service": {
    "name": "greetings-api",
    "id": "greetings-api-v2",
    "tags": ["v2"],
    "meta": {
      "version": "2"
    },
    "address": "10.5.0.5",
    "port": 8080,
    "connect": {
      "sidecar_service": {
        "port": 19003,
        "check": {
          "name": "Connect Envoy Sidecar",
          "tcp": "10.5.0.5:19003",
          "interval": "10s"
        },
        "proxy": {
        }
      }
    }
  }
}
EOF

$ cat << EOF > consul.d/sidecar-for-greetings-client.json
{
  "service": {
    "name": "greetings-client",
    "id":"greetings-client-v1", 
    "address": "10.5.0.3",
    "port": 8080,
    "connect": {
      "sidecar_service": {
        "port": 19001,
        "check": {
          "name": "Connect Envoy Sidecar",
          "tcp": "10.5.0.3:19001",
          "interval": "10s"
        },
        "proxy": {
          "upstreams": [{
             "destination_name": "greetings-api",
             "local_bind_address": "127.0.0.1",
             "local_bind_port": 5000
          }]
        }
      }
    }
  }
}
EOF
```

ここではそれぞれのサービスとそれに付随するサイドカーをレジストしています。`client`のアプリには`upstreams`を設定し、`greetings-api`にリクエストしています。

<kbd>
  <img src="https://github-image-tkaburagi.s3-ap-northeast-1.amazonaws.com/consul-workshop/l7-1.png">
</kbd>

ここでは、Consul 上は`greetings-api-v1`も`greetings-api-v2`も`greetings-api`というサービス名で登録し、各バージョンをサブセットとして管理することにします。`greetings-api-v1`と`greetings-api-v2`にこの設定の中で`meta.version`というメタデータをセットしていることを確認してください。

それでは次に L7 Traffic Management の設定を行なっていきます。`central`フォルダに設定を作っていきます。

まずは`sevice-defaults`の設定です。これは各サービスに反映させるグローバルな設定です。ここでは HTTP で各サービスをやり取りさせるためプロトコルを HTTP として設定します。

```shell
$ cat << EOF > central/api_service_defaults.hcl
kind     = "service-defaults"
name     = "greetings-api"
protocol = "http"
EOF
```

`kind`が設定の種類、`name`が設定を反映させるサービスで、この二つが最低限必要な設定です。

次にサブセットを定義していきます。そのためには`service_resolver`という機能を使います。

```shell
$ cat << EOF > central/api_service_resolver.hcl
kind = "service-resolver"
name = "greetings-api"

default_subset = "v1"

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

`Service.Meta.version`の値が 1 のものを`greetings-api`の`v1`, 2 のものを`v2`として定義しています。またデフォルトのサブセットは`v1`です。

このサブセットを利用してこの後に作成するルーティングの設定をコントロールします。

最後に`service-routing`の設定です。ここでは HTTP ヘッダーの値を基にアップストーム先を切り替えるように設定していきます。

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
            exact = "true"
          },
        ]
      }
    }

    destination {
      service = "greetings-api"
      service_subset = "v2"
    }

  },
]
```

`routes` の設定で`canary`というヘッダー名に`true`がセットされていると`destination`が`v2`のサブセットになっていることがわかると思います。

## 起動する

ここでコンテナを起動してみましょう。まずは v1 のアプリだけが起動し、v2 のアプリは一旦動作を試してから起動させることにします。

```shell
$ docker-compose up
``` 

起動が完了したら`http://127.0.0.1:8500/ui/dc1/services`にアクセスしてみてください。web と api とそれぞれのサイドカーが起動しているはずです。

まず HTTP ヘッダーに`true`以外の値をセットしてリクエストしてみましょう。

```console
$ curl  127.0.0.1:9090/ --header "canary: false"
Greetings From API -> <200,hi i am v1
```

次に`true`をセットしてみます。

```console
$ curl  127.0.0.1:9090/ --header "canary: true"
{"timestamp":"2019-11-20T03:50:06.726+0000","status":500,"error":"Internal Server Error","message":"503 Service Unavailable","path":"/"}
```

Service Unavailable となるでしょう。これは v2 が起動していないためです。

それでは v2 を起動してみます。

```shell
docker-compose -f docker-compose-v2.yaml up
```

`http://127.0.0.1:8500/ui/dc1/services/greetings-api`にアクセスすると、同一サービスの下に新しいバージョンのインスタンスが立ち上がっています。

それではここに HTTP ヘッダーのルーティングでトラフィックを送ってみます。

```console
$ curl  127.0.0.1:9090/ --header "canary: true"
Greetings From API -> <200,hi i am v2

$ curl  127.0.0.1:9090/ --header "canary: false"
Greetings From API -> <200,hi i am v1
```

上記のように`v1`, `v2`に対してルーティングがなされていることがわかるはずです。今回はヘッダーを例に扱いましたが同様にパスベースやパラメータベースでの L7 ルーティングを行うことができます。

## 参考リンク
* [L7 Traffic Management](https://www.consul.io/docs/connect/l7-traffic-management.html)
* [Config Entries](https://www.consul.io/docs/agent/config_entries.html)
* [Service Defaults](https://www.consul.io/docs/agent/config-entries/service-defaults.html)
* [Service Router](https://www.consul.io/docs/agent/config-entries/service-router.html)
* [Service Resolver](https://www.consul.io/docs/agent/config-entries/service-resolver.html)