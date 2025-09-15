# Sidecar Proxy を導入する

Consul ではサイドカーを導入してアプリケーションからは透過的に TLS 通信などを実現出来ます。Consul のサイドカーは plugable です。Built-in の L4 プロキシ、Envoy やカスタムのサイドカーも選択出来ます。

ここでは Envoy と Built-in のプロキシを使ってみましょう。

Consul を起動してサービスを起動しておきましょう。

```shell
$ pkill consul
$ cd consul-workshop
$ DIR=$(pwd)
$ mkdir consul-config-sidecar
$ cat << EOF > consul-config-sidecar/consul-config.hcl
datacenter = "dc1"
data_dir  = "${DIR}/sidecar-car-data"
server = true

bootstrap_expect = 1
ui_config{
  enabled = true
}

bind_addr   = "127.0.0.1"
client_addr = "127.0.0.1"

connect {
  enabled = true
}
EOF
```

```shell
$ consul agent -server \
-config-dir=${DIR}/consul-config-sidecar
```

## Built-in Proxy を利用する

Consul では Built-in の Proxy が用意されており追加で他のソフトウェアをインストールすることなくサイドカーを導入できます。Built-in の Proxy は L4 プロキシのため L7 の機能は提供されていません。

プロキシを導入するときは Service Definition の中に定義していきます。定義には JSON か HCL を使います。定義の方法は二つあります。

* 独立したファイルで設定を定義する
* サービスの設定にネストして設定を定義する

ここでは後者の方法で試してみます。ダミーのサービスを一つ Consul に登録するための定義を作成します。実際のサービスは動いていません。`| dummy-1 -> Sidecar | -> | Sidecar -> dummy-2 |`のようなトラフィックの流れを作っていきます。

```shell
$ cat << EOF > consul-config-sidecar/dummy-1.json
{
  "service": {
    "name": "dummy-1",
    "port": 1234
  }
}
EOF
```

```shell
$ cat << EOF > consul-config-sidecar/dummy-2.json
{
  "service": {
    "name": "dummy-2",
    "port": 4321
  }
}
EOF
```

先ほどまでは CLI でサービスを登録しましたが、ここでは設定ファイルから登録を行なっています。これを反映させるために以下のコマンドを実行します。

```shell
$ consul reload
```

`http://127.0.0.1:8500/ui/dc1/services`にアクセスするとサービスが登録されているでしょう。

このサービスのサイドカーを定義していきます。`dummy-1.json`, `dummy-2.json`を以下のように上書きします。

```hcl
$ cat << EOF > consul-config-sidecar/dummy-1.json
{
  "service": {
    "name": "dummy-1",
    "port": 1234,
    "connect": {
      "sidecar_service": {
        "port": 19001,
        "check": {
          "name": "Connect Built-in Sidecar",
          "tcp": "0.0.0.0:19001",
          "interval": "10s"
        },
        "proxy": {
	        "upstreams": [
	          {
	            "destination_name": "dummy-2",
	            "local_bind_port": 9191
	          }
	      ]
        }
      }
    }
  }
}
EOF
```

プロキシがリクエストを受け付けるためのリスナーの設定と Upstream の設定を行なっています。`local_bind_port`の設定はサイドカーに同居しているアプリケーションが`dummy-2`のアプリにリクエストするためのローカルホストのリスナーです。

Proxy を稼働させるために以下のコマンドを別のターミナルを開いて実行します。エラーが出ると思いますが一旦無視してください。

```shell
$ consul reload
$ consul connect proxy -sidecar-for dummy-1
```

次に dummy-2 の定義をしていきます。

```hcl
$ cat << EOF > consul-config-sidecar/dummy-2.json
{
  "service": {
    "name": "dummy-2",
    "port": 4321,
    "connect": {
      "sidecar_service": {
        "port": 19002,
        "check": {
          "name": "Connect Built-in Sidecar",
          "tcp": "0.0.0.0:19002",
          "interval": "10s"
        },
        "proxy": {
        }
      }
    }
  }
}
EOF
```

Proxy を稼働させるために以下のコマンドを別のターミナルを開いて実行します。エラーが出ると思いますが一旦無視してください。

```shell
$ consul reload
$ consul connect proxy -sidecar-for dummy-2
```

これで dummy-1, dummy-2 用のサイドカーが起動しました。`http://127.0.0.1:8500/ui/dc1/services`にブラウザでアクセスするとサービスが起動していることがわかります。

## Envoy を利用する

Envoy を利用する際は`consul connect`のコマンドで Envoy を指定するだけです。Envoy は Consul のバイナリには組み込まれていないのでインストールを行う必要があります。

[こちらの手順](https://www.envoyproxy.io/docs/envoy/latest/start/install)で Envoy をインストールしてください。

既存のサイドカーのプロセスを停止したら、それぞれ別のターミナルで以下のコマンドを実行してください。

```shell
$ consul connect envoy -sidecar-for dummy-1 -admin-bind 127.0.0.1:19001
$ consul connect envoy -sidecar-for dummy-2 -admin-bind 127.0.0.1:19002
```

```shell
$ ps aux | grep envoy
```

Envoy が起動していることがわかるでしょう。

これ以降の章では Envoy や Built-in を使って様々な Sidecar Proxy の機能を試してみたいと思います。


## 参考リンク
* [Proxy Registration](https://www.consul.io/docs/connect/registration.html)
* [Connect Proxies](https://www.consul.io/docs/connect/proxies.html)
* [Consul Connect CLI](https://www.consul.io/docs/commands/connect.html)
* [Consul Envoy Integration](https://www.consul.io/docs/connect/proxies/envoy.html)
