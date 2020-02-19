# Sidecar Proxyを導入する

Consulではサイドカーを導入してアプリケーションからは透過的にTLS通信などを実現出来ます。Consulのサイドカーはplugableです。Built-inのL4プロキシ、Envoyやカスタムのサイドカーも選択出来ます。

ここではEnvoyとBuilt-inのプロキシを使ってみましょう。

Consulを起動してサービスを起動しておきましょう。

```shell
$ pkill consul
$ cd consul-workshop
$ DIR=$(pwd)
$ mkdir consul-config-sidecar
$ cat << EOF > consul-config-sidecar/consul-config.hcl
datacenter = "dc1"

server = true

bootstrap_expect = 1
ui               = true

bind_addr   = "0.0.0.0"
client_addr = "0.0.0.0"

connect {
  enabled = true
}
EOF
```

```shell
$ consul agent -server -bind=127.0.0.1 \
-client=0.0.0.0 \
-data-dir=${DIR}/sidecar-car-data \
-bootstrap-expect=1 -ui \
-config-dir=${DIR}/consul-config-sidecar
```

## Built-in Proxyを利用する

ConsulではBuilt-inのProxyが用意されており追加で他のソフトウェアをインストールすることなくサイドカーを導入できます。Built-inのProxyはL4プロキシのためL7の機能は提供されていません。

プロキシを導入するときはService Definitionの中に定義していきます。定義にはJSONかHCLを使います。定義の方法は二つあります。

* 独立したファイルで設定を定義する
* サービスの設定にネストして設定を定義する

ここでは後者の方法で試してみます。ダミーのサービスを一つConsulに登録するための定義を作成します。実際のサービスは動いていません。`| dummy-1 -> Sidecar | -> | Sidecar -> dummy-2 |`のようなトラフィックの流れを作っていきます。

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

先ほどまではCLIでサービスを登録しましたが、ここでは設定ファイルから登録を行なっています。これを反映させるために以下のコマンドを実行します。

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

プロキシがリクエストを受け付けるためのリスナーの設定とUpstreamの設定を行なっています。`local_bind_port`の設定はサイドカーに同居しているアプリケーションが`dummy-2`のアプリにリクエストするためのローカルホストのリスナーです。

Proxyを稼働させるために以下のコマンドを別のターミナルを開いて実行します。

```shell
$ consul connect proxy -sidecar-for dummy-1
```

次にdummy-2の定義をしていきます。

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

Proxyを稼働させるために以下のコマンドを別のターミナルを開いて実行します。

```shell
$ consul connect proxy -sidecar-for dummy-2
```

これでdummy-1, dummy-2用のサイドカーが起動しました。`http://127.0.0.1:8500/ui/dc1/services`にブラウザでアクセスするとサービスが起動していることがわかります。

## Envoyを利用する

Envoyを利用する際は`consul connect`のコマンドでEnvoyを指定するだけです。EnvoyはConsulのバイナリには組み込まれていないのでインストールを行う必要があります。

[こちらの手順](https://www.envoyproxy.io/docs/envoy/latest/install/install)でEnvoyをインストールしてください。

既存のサイドカーのプロセスを停止したら、それぞれ別のターミナルで以下のコマンドを実行してください。

```shell
$ consul connect envoy -sidecar-for dummy-1 -admin-bind 127.0.0.1:19001
$ consul connect envoy -sidecar-for dummy-2 -admin-bind 127.0.0.1:19002
```

```shell
$ ps aux | grep envoy
```

Envoyが起動していることがわかるでしょう。

これ以降の章ではEnvoyやBuilt-inのを使って様々なSidecar Proxyの機能を試してみたいと思います。


## 参考リンク
* [Proxy Registration](https://www.consul.io/docs/connect/registration.html)
* [Connect Proxies](https://www.consul.io/docs/connect/proxies.html)
* [Consul Connect CLI](https://www.consul.io/docs/commands/connect.html)
* [Consul Envoy Integration](https://www.consul.io/docs/connect/proxies/envoy.html)