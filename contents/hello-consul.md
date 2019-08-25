# 初めてのConsul

ConsulにDevモードという、複雑な設定や準備の必要なく、簡単にConsul機能を試すモードがあります。起動方法は簡単で、Consulのバイナリをダウンロードし、以下のコマンドを叩くだけです。

## Consulのインストール

[こちら](https://www.consul.io/downloads.html)のWebサイトからご自身のOSに合ったものをダウンロードしてください。

パスを通します。以下はmacOSの例ですが、OSにあった手順でvaultコマンドにパスを通します。

```shell
$ mv /path/to/consul /usr/local/bin
$ chmod +x /usr/local/bin/consul
```
新しい端末を立ち上げ、Vaultのバージョンを確認します。

```console
$ consul --version
Consul v1.5.1
```

これでインストールは完了です。

## ConsulをDevモードで触ってみる

次にVaultサーバを立ち上げ、ConsulのService Discoveryの機能を試してみます。

```shell
consul agent -dev
```

`consul agent`コマンドは、Consulサーバーやクライエントを起動する際に使われます。 通常は様々な設定をコマンドラインオプションや設定ファイルで行いますが、`-dev`ではそのあたりの設定が簡素化されます。あくまでもデモ用途や機能検証などのためですので、本番環境などには推奨しません。

それでは上記のコマンドを叩いてみましょう。

```console
$ consul agent -dev
==> Starting Consul agent...
==> Consul agent running!
           Version: 'v1.5.1+ent'
           Node ID: 'a48c7355-21ea-b48e-5f13-042f2ed599ce'
         Node name: 'masa-mackbook.local'
        Datacenter: 'dc1' (Segment: '<all>')
            Server: true (Bootstrap: false)
       Client Addr: [127.0.0.1] (HTTP: 8500, HTTPS: -1, gRPC: 8502, DNS: 8600)
      Cluster Addr: 127.0.0.1 (LAN: 8301, WAN: 8302)
           Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false

==> Log data will now stream in as it occurs:

    2019/08/20 17:35:45 [DEBUG] agent: Using random ID "a48c7355-21ea-b48e-5f13-042f2ed599ce" as node ID
    2019/08/20 17:35:45 [WARN] agent: Node name "masa-mackbook.local" will not be discoverable via DNS due to invalid characters. Valid characters include all alpha-numerics and dashes.
    2019/08/20 17:35:45 [DEBUG] tlsutil: Update with version 1
    2019/08/20 17:35:45 [DEBUG] tlsutil: OutgoingRPCWrapper with version 1
    2019/08/20 17:35:45 [DEBUG] tlsutil: IncomingRPCConfig with version 1
    2019/08/20 17:35:45 [DEBUG] tlsutil: OutgoingRPCWrapper with version 1
    2019/08/20 17:35:45 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:a48c7355-21ea-b48e-5f13-042f2ed599ce Address:127.0.0.1:8300}]
    2019/08/20 17:35:45 [INFO] raft: Node at 127.0.0.1:8300 [Follower] entering Follower state (Leader: "")
    2019/08/20 17:35:45 [INFO] serf: EventMemberJoin: masa-mackbook.local.dc1 127.0.0.1
    2019/08/20 17:35:45 [INFO] serf: EventMemberJoin: masa-mackbook.local 127.0.0.1
    2019/08/20 17:35:45 [INFO] consul: Adding LAN server masa-mackbook.local (Addr: tcp/127.0.0.1:8300) (DC: dc1)
    2019/08/20 17:35:45 [INFO] consul: Handled member-join event for server "masa-mackbook.local.dc1" in area "wan"
    2019/08/20 17:35:45 [DEBUG] agent/proxy: managed Connect proxy manager started
    2019/08/20 17:35:45 [INFO] agent: Started DNS server 127.0.0.1:8600 (tcp)
    2019/08/20 17:35:45 [INFO] agent: Started DNS server 127.0.0.1:8600 (udp)
    2019/08/20 17:35:45 [INFO] agent: Started HTTP server on 127.0.0.1:8500 (tcp)
    2019/08/20 17:35:45 [INFO] agent: started state syncer
    2019/08/20 17:35:45 [INFO] agent: Started gRPC server on 127.0.0.1:8502 (tcp)
    2019/08/20 17:35:46 [WARN] raft: Heartbeat timeout from "" reached, starting election
    2019/08/20 17:35:46 [INFO] raft: Node at 127.0.0.1:8300 [Candidate] entering Candidate state in term 2
    2019/08/20 17:35:46 [DEBUG] raft: Votes needed: 1
    2019/08/20 17:35:46 [DEBUG] raft: Vote granted from a48c7355-21ea-b48e-5f13-042f2ed599ce in term 2. Tally: 1
    2019/08/20 17:35:46 [INFO] raft: Election won. Tally: 1
    2019/08/20 17:35:46 [INFO] raft: Node at 127.0.0.1:8300 [Leader] entering Leader state
    2019/08/20 17:35:46 [INFO] consul: cluster leadership acquired
    2019/08/20 17:35:46 [INFO] consul: New leader elected: masa-mackbook.local
    2019/08/20 17:35:46 [INFO] connect: initialized primary datacenter CA with provider "consul"
    2019/08/20 17:35:46 [DEBUG] consul: Skipping self join check for "masa-mackbook.local" since the cluster is too small
    2019/08/20 17:35:46 [INFO] consul: member 'masa-mackbook.local' joined, marking health alive
    2019/08/20 17:35:46 [DEBUG] agent: Skipping remote check "serfHealth" since it is managed automatically
    2019/08/20 17:35:46 [INFO] agent: Synced node info
    2019/08/20 17:35:48 [DEBUG] agent: Skipping remote check "serfHealth" since it is managed automatically
    2019/08/20 17:35:48 [DEBUG] agent: Node info in sync
```

様々な情報が出力され、Consulサーバーが起動しました。Consulはプロセスとして走り続けますので、別のターミナルを開いてください。

Consulはデフォルトで以下のアドレスとポートが使われます。


| IP address      | Port | 用途 |
| --------------- |----- | --- | 
| Client address  | 8500 | HTTP APIのインターフェース | 
| Client address  | 8501 | HTTPS APIのインターフェース|  
| Client address  | 8502 | gRPC APIのインターフェース |  
| Client address  | 8600 | DNSのインターフェース |  
| Cluster address | 8300 | サーバー間のRPC用 | 
| Cluster address | 8301 | LAN Serf | 
| Cluster address | 8302 | WAN Serf | 


Client addressは、Consulの機能にアクセスしたいクライアントとの通信に使われ、Cluster addressはConsulサーバー間の通信に使用されます。
ポートについてのより詳細なドキュメントは[こちら](https://www.consul.io/docs/install/ports.html)にあります。

それでは以下のコマンドを打ってみましょう。

```console
$ consul members
Node                 Address         Status  Type    Build      Protocol  DC   Segment
masa-mackbook.local  127.0.0.1:8301  alive   server  1.5.1+ent  2         dc1  <all>
```

`consul members`はこのクラスタにJoinしているサーバーやクライアントを表示します。現時点は一つのサーバーだけが存在しています。


## サービスの登録

Consulの持つ機能の一つにService Registration（サービス登録）やService Discoveryがあります。詳細については、別のワークショップでカバーしますが、ここで簡単に試してみましょう。

サービスの登録方法はいくつかあるのですが、ここでは簡単なCLIからの登録をやってみます。

まず、webというサービスがあり、IPアドレスが10.0.0.10のノードポート8080番で動いてるとします。ちなみに、ここでは実際にこのサービスが動いているかは気にしません。このサービスを登録するには以下のコマンドを打ちます。

```console
$ consul services register -name=web -address=10.0.0.10 -port=8080
Registered service: web
```

これでサービスが登録されました。以下のコマンドで登録されているサービスの一覧が出力できます。

```console
$ consul catalog services
consul
web
```

ちゃんと`web`とサービスが登録されています。

## Service Discovery

登録したサービスはいくつかの方法でDiscoverできます。

* DNS interface
* API
* CLI

ここでは`dig`コマンドによるDNS queryを使ってみましょう。ConsulのDNSはポート8600番なので、そこに対してQueryを発行します。また、サービス名は`<service名>.service.consul`という形式でLookupできます。


```console
$ dig @127.0.0.1 -p 8600 web.service.consul

; <<>> DiG 9.10.6 <<>> @127.0.0.1 -p 8600 web.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 28643
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web.service.consul.		IN	A

;; ANSWER SECTION:
web.service.consul.	0	IN	A	10.0.0.10

;; ADDITIONAL SECTION:
web.service.consul.	0	IN	TXT	"consul-network-segment="

;; Query time: 0 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Tue Aug 20 20:01:30 JST 2019
;; MSG SIZE  rcvd: 99
```

登録した通り`10.0.0.10`が返ってきました。
ここでは、存在していないサービスを登録してService discoveryを試しましたが、現実世界においては、確実に存在しているサービスを登録する必要があります。また、そのサービスが正常に稼働しているかをチェックするHealth checkも行う必要があります。ConsulによるHealth checkや他のサービス登録方法などは別のワークショップで行います。

## 通常モードでConsulを起動する

次に`dev`モードではなく通常モードでConsulを起動してみましょう。起動には`consul`コマンドの引数に設定を記述するかjsonや`HashiCorp Configuration Language`(HCL)でファイルとして記述するなどの方法があります。

ここでは引数に設定してみます。`-data-dir`の値はご自身の環境に合わせて好きな場所をして下さい。devモードで起動しているターミナルを止めて次を実行します。

```shell
$ consul agent -server -bind=127.0.0.1 \
-client=127.0.0.1 \
-data-dir=/Users/kabu/hashicorp/consul/localdata \
-bootstrap-expect=1 -ui
```

同様にConsulが起動するでしょう。最後にWebブラウザにアクセスしてみます。`http://127.0.0.1:8500`にアクセスしてみましょう。

`Services`のタブにConsulと表示され、`Nodes`のタブにご自身のラップトップが登録されていることがわかるでしょう。

以降の章ではこの環境を使って実際の機能を試してみます。

## 参考リンク
* [Consul Command](https://www.consul.io/docs/commands/index.html)
* [Consul Agent](https://www.consul.io/docs/agent/basics.html)
* [Consul Configuration](https://www.consul.io/docs/agent/options.html)