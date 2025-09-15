# Consul Service Mesh

Consul では Service Discovery 以外にも Service Mesh を実現するための様々な機能が用意されています。

* L7 Traffic Management
* Security Policies
* Service Mesh Gateway 
* Proxy
* Observability

などが代表的な機能です。

## Intentions を使った Security Policy

まずは Intentions を利用した Consul の Firewall 機能を試してみましょう。

アプリのイメージは以下の通りです。

<kbd>
  <img src="https://github-image-tkaburagi.s3.ap-northeast-1.amazonaws.com/consul-workshop/intentions-1.png">
</kbd>

### 事前準備

GitHub からレポジトリをクローンします。

```shell
$ pkill consul
$ cd consul-workshop
$ git clone https://github.com/tkaburagi/consul-intentions-demo
```

`docker-compose.yml`のファイルを見てください。`japanapp`,`corpapp`,`hashiapp`,`hashicorpjapanapp`の四つのアプリが定義されており、それぞれのコンテナに Envoy が Sidecar Proxy として同居しています。詳しくは説明で補足します。

トラフィックの流れとしては図の通りで`japanapp`が`Japan`の文字列を返し、`corpapp`が`japanapp`の結果に`Corp`の文字列を Append して`Corp Japan`を返します。そして`hashiapp`が`Hashi`を返し、`hashicorpjapanapp`が`hashiapp`と`corpapp`にリクエストして結果の文字列を連結させ`HashiCorp Japa`をクライアントに返しています。

### サイドカーの設定

この流れをまずは Sidecar Proxy の設定を作っていきます。

```shell
$ cd consul-intentions-demo
$ mkdir consul.d consul_data
```

この中に 4 つのファイルを作っていきます。それぞれがサイドカーの設定となります。

`sidecar-hashicorpjapanapp.json`

```json
cat << EOF > consul.d/sidecar-hashicorpjapanapp.json
{
  "service": {
    "name": "hashicorpjapanapp",
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
          "upstreams": [
            {
             "destination_name": "hashiapp",
             "local_bind_address": "127.0.0.1",
             "local_bind_port": 5000
           },
            {
             "destination_name": "corpapp",
             "local_bind_address": "127.0.0.1",
             "local_bind_port": 5001
           }
         ]
        }
      }
    }
  }
}
EOF
```

`hashiapp`と`corpapp`を Upstream の Destination として設定しています。Proxy によりこれらのサービスにリクエストが振られます。

`sidecar-hashiapp.json`

```json
cat << EOF > consul.d/sidecar-hashiapp.json
{
  "service": {
    "name": "hashiapp",
    "address": "10.5.0.4",
    "port": 8080,
    "connect": {
      "sidecar_service": {
        "port": 19001,
        "check": {
          "name": "Connect Envoy Sidecar",
          "tcp": "10.5.0.4:19001",
          "interval": "10s"
        },
        "proxy": {
          "upstreams": [
         ]
        }
      }
    }
  }
}
EOF
```

`hashiapp`は Upstream は行いませんので Healtcheck のみの設定です。

`sidecar-corpapp.json`

```json
cat << EOF > consul.d/sidecar-corpapp.json
{
  "service": {
    "name": "corpapp",
    "address": "10.5.0.5",
    "port": 8080,
    "connect": {
      "sidecar_service": {
        "port": 19001,
        "check": {
          "name": "Connect Envoy Sidecar",
          "tcp": "10.5.0.5:19001",
          "interval": "10s"
        },
        "proxy": {
          "upstreams": [
            {
             "destination_name": "japanapp",
             "local_bind_address": "127.0.0.1",
             "local_bind_port": 5000
           }
         ]
        }
      }
    }
  }
}
EOF
```

`corpapp`は説明の通り`japanapp`から文字列を取得するため Upsetream の設定を行なっています。

`sidecar-japanapp.json`

```json
cat << EOF > consul.d/sidecar-japanapp.json
{
  "service": {
    "name": "japanapp",
    "address": "10.5.0.6",
    "port": 8080,
    "connect": {
      "sidecar_service": {
        "port": 19001,
        "check": {
          "name": "Connect Envoy Sidecar",
          "tcp": "10.5.0.6:19001",
          "interval": "10s"
        },
        "proxy": {
          "upstreams": [
         ]
        }
      }
    }
  }
}
EOF
```

`japanapp`も Upstream 先はありません。サイドカーの設定は以上です。

`sidecar-unintentionalapp.json`

```json
cat << EOF > consul.d/sidecar-unintentionalapp.json
{
  "service": {
    "name": "unintentionalapp",
    "address": "10.5.0.7",
    "port": 8080,
    "connect": {
      "sidecar_service": {
        "port": 19001,
        "check": {
          "name": "Connect Envoy Sidecar",
          "tcp": "10.5.0.7:19001",
          "interval": "10s"
        },
        "proxy": {
          "upstreams": [
            {
             "destination_name": "hashiapp",
             "local_bind_address": "127.0.0.1",
             "local_bind_port": 5000
           },
            {
             "destination_name": "corpapp",
             "local_bind_address": "127.0.0.1",
             "local_bind_port": 5001
           }
         ]
        }
      }
    }
  }
}
EOF
```

これは実態は`hashicorjapanapp`と同じですが、別のサービスとして扱います。最後に利用するので今は気にしないでください。

### アプリのデプロイ

このコンフィグを使って Docker コンテナを立ち上げてみます。

**sudo が必要な方は`run-sudo.sh`を実行してください。**

```shell
$ ./run.sh
```

上で作った各コンフィグレーションは Docker Compose の中で各コンテナにマウントされ、コンフィグとして扱われるように設定されています。

GUI からこのような状態になっていれば起動は成功です。
<kbd>
  <img src="https://github-image-tkaburagi.s3.ap-northeast-1.amazonaws.com/consul-workshop/intentions-s.png">
</kbd>


Docker Compose の起動が完了したら各アプリをテストしてみましょう。

```console
$ curl http://127.0.0.1:3030
Japan

$ curl http://127.0.0.1:2020
Corp Japan

$ curl http://127.0.0.1:1010
Hashi

$ curl http://127.0.0.1:8080
HashiCorp Japan

$ curl http://127.0.0.1:9090
HashiCorp Japan
```

今の状態だとこの Consul 上にある全てのサービスからどの API にも API にもアクセス出来てしまうためセキュアではありません。通常ファイヤーウォールなどを構築してセキュリティグループを定義していきますが、Consul ではサービスベースの制御が可能です。

次に`Intentions`を使ってサービスベースで FW を組んでいきます。

### Intentions を使ったアクセス制御

まず全てのサービスをデフォルトでリクエストを受け付けないように設定します。

`consul intention create -deny/-allow <SOURCE_SERVICE> <DEST_SERVICE>`という形で簡単に定義出来ます。

```shell
$ consul intention create -deny '*' hashiapp
```

この状態だと`hashiapp`にアクセス不可能なため、`hashicorpjapanapp`からリクエストするとエラーになるはずです。

```console
$ curl http://127.0.0.1:8080
{"timestamp":"2019-09-18T06:46:22.716+0000","status":500,"error":"Internal Server Error","message":"I/O error on GET request for \"http://127.0.0.1:5000/\": Unexpected end of file from server; nested exception is java.net.SocketException: Unexpected end of file from server","path":"/"}
```

一方、`corp` - `japan`間にはなんの設定もされていません。

```console
$ curl http://127.0.0.1:2020
Corp Japan
```

このようにアクセス出来るでしょう。

残りの`-deny`を設定します。

```shell
$ consul intention create -deny '*' corpapp
$ consul intention create -deny '*' japanapp
$ consul intention create -deny '*' hashicorpjapanapp
```

```console
$ curl http://127.0.0.1:2020
{"timestamp":"2019-09-18T06:50:31.130+0000","status":500,"error":"Internal Server Error","message":"I/O error on GET request for \"http://127.0.0.1:5000/\": Unexpected end of file from server; nested exception is java.net.SocketException: Unexpected end of file from server","path":"/"}%
```

設定通りエラーとなりました。

次にアプリが正常に動くよう Allow の設定します。出来るだけ下記の手順を見ずに図を見ながら設定してみてください。`consul intention create -deny/-allow <SOURCE_SERVICE> <DEST_SERVICE>`です。

完成形はこのようになります。

<kbd>
  <img src="https://github-image-tkaburagi.s3.ap-northeast-1.amazonaws.com/consul-workshop/intentions-2.ong.png">
</kbd>

```shell
$ consul intention create -allow corpapp japanapp
$ consul intention create -allow hashicorpjapanapp corpapp
$ consul intention create -allow hashicorpjapanapp hashiapp
```

設定が終わったらテストしてみましょう。

```console
$ curl http://127.0.0.1:8080
HashiCorp Japan
```

きちんと End to End でリクエストが処理されました。最後に意図しないサービスを立ち上げ、そこから各サービスにアクセスが出来ないことを確認します。


`unintentionalapp`にアクセスしてみます。このアプリはアプリの実態と Upstream の設定は`hashicorpjapanapp`と同じですが Intentions によりバックエンドの API サービスへのアクセスは許可されていません。

`9090`でアクセスできます。

```console
$ curl http://127.0.0.1:9090
{"timestamp":"2019-09-18T08:50:15.928+0000","status":500,"error":"Internal Server Error","message":"I/O error on GET request for \"http://127.0.0.1:5000/\": Unexpected end of file from server; nested exception is java.net.SocketException: Unexpected end of file from server","path":"/"}
```

エラーとなり許可しないサービス以外からはアクセスできないことがわかります。

このように Intentions を利用するとサービスベースで ACL を設定でき、物理的なロケーションの resolve は Consul に任せることができます。

最後に`Ctr+C`で抜けて全コンテナを停止しておきましょう。

```shell
$ docker-compose down
```

## 参考リンク
* [Intentions](https://www.consul.io/docs/connect/intentions.html)
* [Proxy](https://www.consul.io/docs/connect/proxies.html)
* [CA Management](https://www.consul.io/docs/connect/ca.html)
