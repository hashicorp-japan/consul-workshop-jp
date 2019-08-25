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


### 事前準備

この手順を完了させるためには`socat`と`netcat`というツールが必要です。Socatを使ってエコーサーバを立ち上げ、Netcatで接続をしてデータの送受信ををします。

* socat
	* macOS: [インストール方法](http://macappstore.org/socat/)
	* Windows: [インストール方法](http://pioneertools.blogspot.com/2018/01/how-to-install-socat-network-utility.html)
* netcat
	* macOS: [インストール方法](http://macappstore.org/netcat/)
	* Windows: [インストール方法](https://stackoverflow.com/questions/40944370/how-to-install-netcat-in-windows)

### Sidecarの導入

Sidecar間はTLS通信となり通常証明書が必要です。Consulには[証明書管理の機能](https://www.consul.io/docs/connect/ca.html)が内包されており、`dev`モードではこの機能を利用して起動時に証明書を発行しデフォルトでTLS通信が可能です。そのため一度Consulを止め`dev`モードで起動し直します。

```shell
$ consul agent -dev \
-data-dir=/Users/kabu/hashicorp/consul/localdata \
-config-dir=/Users/kabu/workspace/consul.d
```

別ターミナルで`socat`を起動します。

```shell
$ socat -v tcp-l:8181,fork exec:"/bin/cat"
```

Consulの設定ファイルを追加して設定をリロードします。

```shell
$ cat << EOF > /path/to/workspace/consul.d/socat.json 
{
  "service": {
    "name": "socat",
    "port": 8181,
    "connect": { "sidecar_service": {} }
  }
}
EOF

$ consul reload
```

また別ターミナルを立ち上げSocat用のSidecarを起動します。

```shell
$ consul connect proxy -sidecar-for socat
```

このサービスとやり取りをする別のサービスを立ち上げます。

```shell
$ cat << EOF > /path/to/workspace/consul.d/web.json 
{"service": {
    "name": "web",
    "port": 8080,
    "connect": {
      "sidecar_service": {
        "proxy": {
          "upstreams": [{
             "destination_name": "socat",
             "local_bind_port": 9191
          }]
        }
      }
    }
  }
}
EOF

$ consul reload
```

また別ターミナルを立ち上げWeb用のSidecarを起動します。

```shell
$ consul connect proxy -sidecar-for web
```

GUIの`Services`タブで状況を確認しておきましょう。

```
(8080/Web/Sidecar/9191) <-TLS-> (8181/Sidecar/Socat)
```

の用なトポロジーになっています。Sidecar(:9191)経由でSocatにアクセスしてみましょう。

```cosole
$ nc 127.0.0.1 9191
hi
hi
```

次にこの両者の接続にIntentionsを利用してアクセス制御の設定を行います。

```console
$ consul intention create -deny web socat
Created: web => socat (deny)
```

設定はこれだけです。実際には優先度やワイルドカードを利用して接続の制御を設定していきます。

設定を確認してみましょう。GUIの`Intentions`タブからも確認することが出来ます。

```console
$ consul intention get web socat
Source:       web
Destination:  socat
Action:       deny
ID:           de5c0376-3e69-b78a-1b88-cdb3d6008f3c
Created At:   Monday, 26-Aug-19 00:01:14 JST
```

それでは再度Socatにアクセスしてみます。

```console
$ nc 127.0.0.1 9191
```

メッセージが出ないのでわかりにくいですがアクセス制御で弾かれて接続が不可能になっています。
