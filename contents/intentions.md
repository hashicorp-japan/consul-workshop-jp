# Consul Service Mesh

ConsulではService Discovery以外にもService Meshを実現するための様々な機能が用意されています。

* L7 Traffic Management
* Security Policies
* Service Mesh Gateway 
* Proxy
* Observability

などが代表的な機能です。

## Intentionsを使ったSecurity Policy

まずはIntentionsを利用したConsulのFirewall機能を試してみましょう。

アプリのイメージは以下の通りです。

<kbd>
  <img src="https://github-image-tkaburagi.s3.ap-northeast-1.amazonaws.com/consul-workshop/intentions-1.png">
</kbd>

### 事前準備

GitHubからレポジトリをクローンします。

```shell
$ cd consul-workshop
$ git clone https://github.com/tkaburagi/consul-intentions-demo
```

`docker-compose.yml`のファイルを見てください。`japanapp`,`corpapp`,`hashiapp`,`hashicorpjapanapp`の四つのアプリが定義されており、それぞれのコンテナにEnvoyがSidecar Proxyとして同居しています。詳しくは説明で補足します。

トラフィックの流れとしては図の通りで`japanapp`が`Japan`の文字列を返し、`corpapp`が`japanapp`の結果に`Corp`の文字列をAppendして`Corp Japan`を返します。そして`hashiapp`が`Hashi`を返し、`hashicorpjapanapp`が`hashiapp`と`corpapp`にリクエストして結果の文字列を連結させ`HashiCorp Japa`をクライアントに返しています。

### サイドカーの設定

この流れをまずはSidecar Proxyの設定を作っていきます。

```shell
$ cd consul-intentions-demo
$ mkdir consul.d
$ cd consul.d
```

この中に4つのファイルを作っていきます。それぞれがサイドカーの設定となります。

`sidecar-hashicorpjapanapp.json`

```json
cat << EOF > sidecar-hashicorpjapanapp.json`
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

`hashiapp`と`corpapp`をUpstreamのDestinationとして設定しています。Proxyによりこれらのサービスにリクエストが振られます。

`sidecar-hashiapp.json`

```json
cat << EOF > idecar-hashiapp.json
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

`hashiapp`はUpstreamは行いませんのでHealtcheckのみの設定です。

`sidecar-corpapp.json`

```json
cat << EOF > sidecar-corpapp.json
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

`corpapp`は説明の通り`japanapp`から文字列を取得するためUpsetreamの設定を行なっています。

`sidecar-japanapp.json`

```json
cat << EOF > sidecar-japanapp.json
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

`japanapp`もUpstream先はありません。サイドカーの設定は以上です。

`sidecar-unintentionalapp.json`

```json
cat << EOF > sidecar-unintentionalapp.json
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

このコンフィグを使ってDockerコンテナを立ち上げてみます。

```shell
$ ./run.sh
```

上で作った各コンフィグレーションはDocker Composeの中で各コンテナにマウントされ、コンフィグとして扱われるように設定されています。

GUIからこのような状態になっていれば起動は成功です。
<kbd>
  <img src="https://github-image-tkaburagi.s3.ap-northeast-1.amazonaws.com/consul-workshop/intentions-s.png">
</kbd>


Docker Composeの起動が完了したら各アプリをテストしてみましょう。

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

今の状態だとこのConsul上にある全てのサービスからどのAPIにもAPIにもアクセス出来てしまうためセキュアではありません。通常ファイヤーウォールなどを構築してセキュリティグループを定義していきますが、Consulではサービスベースの制御が可能です。

次に`Intentions`を使ってサービスベースでFWを組んでいきます。

### Intentionsを使ったアクセス制御

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

次にアプリが正常に動くようAllowの設定します。出来るだけ下記の手順を見ずに図を見ながら設定してみてください。`consul intention create -deny/-allow <SOURCE_SERVICE> <DEST_SERVICE>`です。

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

きちんとEnd to Endでリクエストが処理されました。最後に意図しないサービスを立ち上げ、そこから各サービスにアクセスが出来ないことを確認します。


`unintentionalapp`にアクセスしてみます。このアプリはアプリの実態とUpstreamの設定は`hashicorpjapanapp`と同じですがIntentionsによりバックエンドのAPIサービスへのアクセスは許可されていません。

`9090`でアクセスできます。

```console
$ curl http://127.0.0.1:9090
{"timestamp":"2019-09-18T08:50:15.928+0000","status":500,"error":"Internal Server Error","message":"I/O error on GET request for \"http://127.0.0.1:5000/\": Unexpected end of file from server; nested exception is java.net.SocketException: Unexpected end of file from server","path":"/"}
```

エラーとなり許可しないサービス以外からはアクセスできないことがわかります。

このようにIntentionsを利用するとサービスベースでACLを設定でき、物理的なロケーションのresolveはConsulに任せることができます。

## 参考リンク
* [Intentions](https://www.consul.io/docs/connect/intentions.html)
* [Proxy](https://www.consul.io/docs/connect/proxies.html)
* [CA Management](https://www.consul.io/docs/connect/ca.html)
