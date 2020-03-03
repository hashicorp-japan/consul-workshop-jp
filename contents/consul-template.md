# Consul Template

Consul TemplateはこのKVを応用したトリガー処理の一つです。テンプレートに更新内容の雛形を定義しておき、KVに更新が出た際にそれを指定し、その内容を指定したファイルに埋め込むことのできる機能です。

簡単な例でまずは試してみましょう。先にインストールを行います。

[こちら](https://github.com/hashicorp/consul-template#installation)の[ダウンロードリンク](https://releases.hashicorp.com/consul-template/)からご自身のOSにあったものをダウンロード、解凍しパスを通してください。

次にテンプレートを用意します。

```shell
$ cat <<EOF > my-first-consul.tpl
Updated Value -> {{ key "/my-first-kv/consulis" }}
EOF
```

Workspace用のディレクトリで以下を実行します。プロンプトは戻らないのが正しいです。

```shell
$ consul-template -template="my-first-consul.tpl:consulis.txt"
```

`my-first-consul.tpl`の更新を`consulis.txt`のファイルに書き込んでいます。この状態で値を更新してみましょう。

```shell
$ consul kv put my-first-kv/consulis super-useful
```

`consulis.txt`というファイルができているはずなので中身を確認します。

```console
$ cat workspace/consulis.txt
Updated Value -> super-useful
```

テンプレートで指定した`{{ key "/my-first-kv/consulis" }}`のキーの更新内容がファイルに書き込まれていることがわかります。

再度更新してみましょう。

```console
$ consul kv put my-first-kv/consulis made-by-hashicorp
$ cat workspace/consulis.txt
Updated Value -> made-by-hashicorp
```

同じ様に更新が確認できるでしょう。

### Consul Templateのユースケース

さて、Consul Templateを少し試してみましたがどの様なユースケースがあるのでしょうか？実はユースケースはかなり多岐に渡ります。

Consul TemplateはConsul配下のデータセンター、ノードやサービスの情報を収集することができ、これを利用することでConsulのサービス構成に何らかの変更があった際に特定のファイルを自動で生成することができます。

まずは環境の情報の取得をしてみましょう。

```shell
cat <<EOF > all-services.tpl
{{ range datacenters }}
{{ . }}{{ end }}

{{ range nodes }}
{{.Address}}{{ end }}

{{range services}}# {{.Name}}{{range service .Name}}
{{.Address}}{{end}}

{{end}}
EOF
```

これを使ってconsul-templateを実行してみましょう。

```shell
$ consul-template -template="all-services.tpl:output/all-services.txt" -once
```

`-once`オプションをつけることで一度きりの実行が可能です。

```console
$ cat output/all-services.txt
dc1
dc2

127.0.0.1
192.168.0.1

# consul
127.0.0.1

# greetings-api
127.0.0.1
127.0.0.1

# greetings-api-sidecar-proxy

# greetings-client
127.0.0.1

# greetings-client-sidecar-proxy

# mysql
127.0.0.1

# nginx
192.168.3.3
127.0.0.1
```

結果は上記の異なるはずですがこの環境に関する情報がダンプ出来たはずです。

### Nginxの設定ファイルを自動生成する

さて、ここまでconsul-templateの基本を見てきましたが、この機能を使ってConsul内のサービス構成の更新情報を取得しHAProxyやNginxの設定ファイルを自動更新して反映させるといった運用を自動化できます。またVaultと連携をさせることでVaultから発行した証明書をLBにセットするなども自動で可能です。

ここではNginxのLoadBalancer機能で試してみましょう。

ここでは先ほど立ち上げた`Foo`と`Bar`のコンテナのロードバランサーとしてローカルで起動するNginxを使います。

>Nginxが入っていない方はこちらからインストールしてください。
>* [Linux](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/)
>* Mac: `brew install nginx`
>* [Windows](http://nginx.org/en/docs/windows.html)

それぞれのOSにあった起動方法で起動します。以下はmacOSの例です。

```shell
$ sudo nginx
```

それぞれのOSでNginxの設定ファイルのディレクトリを探して下さい。Macの場合はこちらです。

```shell
$ cd /usr/local/etc/nginx
```

ここに以下のファイルを作成します。

```shell
cat <<EOF > nginx.conf.cfvmpl
http {
	upstream backend {
	{{ range service "nginx" }}
	  server {{ .Address }}:{{ .Port }};
	{{ end }}
	}

    server {
        listen       80;
        server_name  localhost;

	   location / {
	      proxy_pass http://backend;
	   }
	}
}
events{
}
EOF
```

これがConsul Templateファイルです。次にConsul Templateの設定ファイルをworkspaceディレクトリに作ります。

```shell
cat <<EOF > path/to/workspace/consul-template-config.hcl
consul {
  address = "127.0.0.1:8500"
  retry {
  enabled = true
  attempts = 12
  backoff = "250ms"
  }}

template {
  source      = "/usr/local/etc/nginx/nginx.conf.ctvmpl"
  destination = "/usr/local/etc/nginx/nginx.conf"
  perms = 0600
  command = "sudo nginx -s reload"
}
EOF
```


ローカルのコンサルの構成情報を250msに一度取りに行きテンプレートから`nginx.conf`を生成し、nginxを再起動しています。`command`の再起動コマンドは各OSに合わせてください。


```shell
$ cat nginx.conf
```

Nginxの設定ファイルがデフォルトであることを確認してください。次にConsul Templateを起動します。

```shell
$ consul-template -config=consul-template-config.hcl
```

再度設定ファイルを確認します。

```console
$ cat nginx.conf

http {
	upstream backend {

	  server 127.0.0.1:9090;

	  server 127.0.0.1:8080;

	}

    server {
        listen       80;
        server_name  localhost;

	   location / {
	      proxy_pass http://backend;
	   }
	}
}
events{
}
```

Consul Templateにより以上の様に書き換わっているはずです。

アクセスしてみましょう。

```console
$ curl localhost:80
<html>
<body>
<h1>Hello Consule From Foo Container</h1>
</body>
</html>

$ curl localhost:80
<html>
<body>
<h1>Hello Consule From Bar Container</h1>
</body>
</html>
```

複数回リクエストすると二つのコンテナにアクセス出来るでしょう。次に新しいコンテナを一つ起動させます。

先ほどの`consul-workshop/assets`内にある`index.html`の`Bar`の文字列を`Hoge`に変更して起動します。

```shell
$ docker build -t nginx-image .
$ docker run --name nginx-hoge -d -p 7070:80 nginx-image
```

Consulにサービス登録します。

```shell
$ consul services register \
-name=nginx \
-id=nginx-hoge \
-address=127.0.0.1 \
-port=7070 \
-tag=nginx
```

登録が成功したら、設定ファイルを見てみましょう。

```console
$ cat nginx.conf

http {
	upstream backend {

	  server 127.0.0.1:9090;

	  server 127.0.0.1:8080;

	  server 127.0.0.1:7070;

	}

    server {
        listen       80;
        server_name  localhost;

	   location / {
	      proxy_pass http://backend;
	   }
	}
}
events{
}
```

即座に設定ファイルが追加されます。念の為再度アプリにアクセスしてみます。

```console
$ curl localhost:80
<html>
<body>
<h1>Hello Consule From Foo Container</h1>
</body>
</html>

$ curl localhost:80
<html>
<body>
<h1>Hello Consule From Bar Container</h1>
</body>
</html>

$ curl localhost:80
<html>
<body>
<h1>Hello Consule From Hoge Container</h1>
</body>
</html>
```

Hogeのコンテナに負荷分散されるはずです。

余裕のある方は`consul services deregister -id=***`コマンドでサービスを一つ除外して同様に設定ファイルとリクエストのレスポンスを確認してみましょう。

このようにConsul Templateを利用するとConsul内にある様々なサービスとそれに関連する外部の設定ファイルを自動で更新同期することが可能となります。またConsul TemplateはVaultとも連携可能でVaultが発行する暗号化のキーや証明書を同様に自動で配布することができます。

## 参考リンク
* [KV](https://www.consul.io/docs/agent/kv.html)
* [Consul Template](https://github.com/hashicorp/consul-template)
