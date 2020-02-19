# Consul Service Discoveryを使ってみる

Consulの機能は多岐に渡りますが、Service Discoveryはコアの機能です。Service Discoeryの機能によりシステムを構成するコンポーネント同士はService名ベースでの接続が可能となります。

ここではサービスをいくつか登録し、実際のDNSやヘルスチェックなどの機能を試してみます。

ここではローカルにNginxを2インスタンス起動させConsulからService Discoveryの機能を試してみます。

まずDockerで二つのインタンスを起動させます。

```shell
$ cd consul-workshop
$ git clone https://github.com/tkaburagi/nginx
$ cd nginx

$ docker-compose up -d
```

起動を確認しておきましょう。

```console
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
109f9530196a        nginx_bar         "nginx -g 'daemon of…"   16 seconds ago      Up 15 seconds       0.0.0.0:9090->80/tcp   nginx-bar
ac31d5eec216        nginx-foo        "nginx -g 'daemon of…"   27 seconds ago      Up 26 seconds       0.0.0.0:8080->80/tcp   nginx_foo
```

curlで両インスタンスからのレスポンスを確認ておきます。

```console
$ curl 127.0.0.1:8080/index.html
<html>
<body>
<h1>Hello Consule From Foo Container</h1>
</body>
```

```console 
$ curl 127.0.0.1:9090/index.html
<html>
<body>
<h1>Hello Consule From Bar Container</h1>
</body>
</html>
```

それではこれらConsulにRegistrationしてみます。

以下のコマンドを実行し、Consulにサービスを登録します。
```shell
$ consul services register \
-name=nginx \
-id=nginx_foo \
-address=127.0.0.1 \
-port=8080 \
-tag=nginx
```

`bar`のコンテナも同様に登録します。

```shell
$ consul services register \
-name=nginx \
-id=nginx_bar \
-address=127.0.0.1 \
-port=9090 \
-tag=nginx
```

Consulのエンドポイントを確認してサービスが登録されていることを確認してみましょう。GUIでも同様のことが出来ます。
```console
$ curl http://127.0.0.1:8500/v1/catalog/service/nginx | jq

[
  {
    "ID": "2db4a816-17fb-3a9d-0a5f-df8739c75915",
    "Node": "Takayukis-MBP",
    "Address": "127.0.0.1",
    "Datacenter": "dc1",
    "TaggedAddresses": {
      "lan": "127.0.0.1",
      "wan": "127.0.0.1"
    },
    "NodeMeta": {
      "consul-network-segment": ""
    },
    "ServiceKind": "",
    "ServiceID": "nginx-1",
    "ServiceName": "nginx",
    "ServiceTags": [
      "nginx"
    ],
    "ServiceAddress": "127.0.0.1",
    "ServiceWeights": {
      "Passing": 1,
      "Warning": 1
    },
    "ServiceMeta": {},
    "ServicePort": 8080,
    "ServiceEnableTagOverride": false,
    "ServiceProxyDestination": "",
    "ServiceProxy": {},
    "ServiceConnect": {},
    "CreateIndex": 547,
    "ModifyIndex": 554
  }
]
```

digで確かめてみます。

```console
$ dig @127.0.0.1 -p 8600 nginx.service.consul. SRV

; <<>> DiG 9.10.6 <<>> @127.0.0.1 -p 8600 nginx.service.consul. SRV
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 3031
;; flags: qr aa rd; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 5
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;nginx.service.consul.		IN	SRV

;; ANSWER SECTION:
nginx.service.consul.	0	IN	SRV	1 1 8080 Takayukis-MBP.node.dc1.consul.
nginx.service.consul.	0	IN	SRV	1 1 9090 Takayukis-MBP.node.dc1.consul.

;; ADDITIONAL SECTION:
Takayukis-MBP.node.dc1.consul. 0 IN	A	127.0.0.1
Takayukis-MBP.node.dc1.consul. 0 IN	TXT	"consul-network-segment="
Takayukis-MBP.node.dc1.consul. 0 IN	A	127.0.0.1
Takayukis-MBP.node.dc1.consul. 0 IN	TXT	"consul-network-segment="

;; Query time: 0 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Sun Aug 25 15:53:01 JST 2019
;; MSG SIZE  rcvd: 251
```

`nginx.service.consul`に対してFooとBarの二つのインスタンスにアクセスできることがわかります。

## HealthCheckの設定

次にHealthCheckの設定を行います。以下のJsonファイルを作成して下さい。

```shell
$ cat << EOF > check_foo.json 
{
  "ID": "nginx_foo",
  "Name": "nginx",
  "Address": "127.0.0.1",
  "Port": 8080,
  "check": {
    "http": "http://127.0.0.1:8080",
    "interval": "10s",
    "timeout": "1s"
   }
}
EOF
```

```shell
$ cat << EOF > check_bar.json 
{
  "ID": "nginx_bar",
  "Name": "nginx",
  "Address": "127.0.0.1",
  "Port": 9090,
  "check": {
    "http": "http://127.0.0.1:9090",
    "interval": "10s",
    "timeout": "1s"
   }
}
EOF
```

`http`で指定したエンドポイントに対して10秒に一度ヘルスチェックを行う設定です。上記の設定を反映させてみます。

```shell
$ curl -X PUT --data-binary @check_foo.json http://127.0.0.1:8500/v1/agent/service/register
$ curl -X PUT --data-binary @check_bar.json http://127.0.0.1:8500/v1/agent/service/register
```

ヘルスチェックの確認をしてみましょう。GUIでも見ることができます。
```shell
$ curl http://127.0.0.1:8500/v1/health/checks/nginx | jq .
```

<details><summary>出力例</summary>
	
```json
[
  {
    "Node": "Takayukis-MBP",
    "CheckID": "service:nginx-bar",
    "Name": "Service 'nginx' check",
    "Status": "passing",
    "Notes": "",
    "Output": "HTTP GET http://127.0.0.1:9090: 200 OK Output: <html>\n<body>\n<h1>Hello Consule From Bar Container</h1>\n</body>\n</html>\n",
    "ServiceID": "nginx-bar",
    "ServiceName": "nginx",
    "ServiceTags": [],
    "Definition": {},
    "CreateIndex": 1265,
    "ModifyIndex": 1292
  },
  {
    "Node": "Takayukis-MBP",
    "CheckID": "service:nginx-foo",
    "Name": "Service 'nginx' check",
    "Status": "passing",
    "Notes": "",
    "Output": "HTTP GET http://127.0.0.1:8080: 200 OK Output: <html>\n<body>\n<h1>Hello Consule From Foo Container</h1>\n</body>\n</html>\n",
    "ServiceID": "nginx-foo",
    "ServiceName": "nginx",
    "ServiceTags": [],
    "Definition": {},
    "CreateIndex": 1306,
    "ModifyIndex": 1307
  }
]
```
</details>

次にfooのコンテナを停止させてみます。

```shell
$ docker rm -f nginx_foo_1
```

再度確認してみます。
```shell
$ curl http://127.0.0.1:8500/v1/health/checks/nginx | jq .
```

<details><summary>出力例</summary>
	
```json
[
  {
    "Node": "Takayukis-MBP",
    "CheckID": "service:nginx-bar",
    "Name": "Service 'nginx' check",
    "Status": "passing",
    "Notes": "",
    "Output": "HTTP GET http://192.168.3.3:9090: 200 OK Output: <html>\n<body>\n<h1>Hello Consule From Bar Container</h1>\n</body>\n</html>\n",
    "ServiceID": "nginx-bar",
    "ServiceName": "nginx",
    "ServiceTags": [],
    "Definition": {},
    "CreateIndex": 1265,
    "ModifyIndex": 1292
  },
  {
    "Node": "Takayukis-MBP",
    "CheckID": "service:nginx-foo",
    "Name": "Service 'nginx' check",
    "Status": "critical",
    "Notes": "",
    "Output": "Get http://192.168.3.3:8080: dial tcp 192.168.3.3:8080: connect: connection refused",
    "ServiceID": "nginx-foo",
    "ServiceName": "nginx",
    "ServiceTags": [],
    "Definition": {},
    "CreateIndex": 1306,
    "ModifyIndex": 1322
  }
]
```
</details>

fooのコンテナの`status`がCriticalに変化しています。この状態でdigるとどうなるでしょうか。

```shell
$ dig @127.0.0.1 -p 8600 nginx.service.consul SRV

; <<>> DiG 9.10.6 <<>> @127.0.0.1 -p 8600 nginx.service.consul. SRV
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 50525
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 3
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;nginx.service.consul.		IN	SRV

;; ANSWER SECTION:
nginx.service.consul.	0	IN	SRV	1 1 9090 Takayukis-MBP.node.dc1.consul.

;; ADDITIONAL SECTION:
Takayukis-MBP.node.dc1.consul. 0 IN	A	127.0.0.1
Takayukis-MBP.node.dc1.consul. 0 IN	TXT	"consul-network-segment="

;; Query time: 0 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Sun Aug 25 22:08:32 JST 2019
;; MSG SIZE  rcvd: 150
```

一つのサーバのみ返ってきており、Consulが停止したサーバを自動で切り離したことがわかります。再起動しましょう。

```shell
$ docker-compose up
```

```console
$ dig @127.0.0.1 -p 8600 nginx.service.consul. SRV

; <<>> DiG 9.10.6 <<>> @127.0.0.1 -p 8600 nginx.service.consul. SRV
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 26653
;; flags: qr aa rd; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 5
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;nginx.service.consul.		IN	SRV

;; ANSWER SECTION:
nginx.service.consul.	0	IN	SRV	1 1 9090 Takayukis-MBP.node.dc1.consul.
nginx.service.consul.	0	IN	SRV	1 1 8080 Takayukis-MBP.node.dc1.consul.

;; ADDITIONAL SECTION:
Takayukis-MBP.node.dc1.consul. 0 IN	A	127.0.0.1
Takayukis-MBP.node.dc1.consul. 0 IN	TXT	"consul-network-segment="
Takayukis-MBP.node.dc1.consul. 0 IN	A	127.0.0.1
Takayukis-MBP.node.dc1.consul. 0 IN	TXT	"consul-network-segment="

;; Query time: 0 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Sun Aug 25 22:07:48 JST 2019
;; MSG SIZE  rcvd: 251
```

正しい状態に戻りました。ConsulのService Registryの機能を使うことでプラットフォームに依存せずサービス名ベースでの相互接続やサービス間のヘルスチェックが可能です。実際のアプリケーションからはConsulを経由して、APIやデータストアなどのシステムコンポーネントにアクセスすることが出来ます。

また、Service RegistryだけでなくService Meshを実現するための様々な機能を備えています。


最後に`Ctr+C`で抜けて全コンテナを停止しておきましょう。

```shell
$ docker-compose down
```

## 参考リンク
* [DNS Interface](https://www.consul.io/docs/agent/dns.html)
* [Health Checks](https://www.consul.io/docs/agent/checks.html)
