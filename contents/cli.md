# Consul CLI を色々と試す

ここでは consul cli の基本的な使い方のガイドを通して Consul の様々な機能を試してみたいと思います。

まずはクラスタ構成が Consul を立ち上げてみます。

```shell
$ curl https://raw.githubusercontent.com/hashicorp/consul/master/demo/docker-compose-cluster/docker-compose.yml > docker-compose.yml

$ docker-compose up 
```

## completion

Consul ではコマンド入力をサポートするための入力補足を用意しています。以下のコマンドでインストールしてみましょう。

```shell
$ consul -autocomplete-install
```

これを利用するとタブでサブコマンドの入力を補完してくれます。

## info

まずは Consul のクラスタ情報の概要を確認するためのコマンドです。

```shell
$ consul info
```

<details><summary>出力例</summary>

```
agent:
	check_monitors = 0
	check_ttls = 0
	checks = 0
	services = 0
build:
	prerelease =
	revision = 944cc710
	version = 1.6.0
consul:
	acl = disabled
	bootstrap = false
	known_datacenters = 1
	leader = false
	leader_addr = 172.27.0.7:8300
	server = true
raft:
	applied_index = 419
	commit_index = 419
	fsm_pending = 0
	last_contact = 38.0769ms
	last_log_index = 419
	last_log_term = 3
	last_snapshot_index = 0
	last_snapshot_term = 0
	latest_configuration = [{Suffrage:Voter ID:9acf7003-b685-e834-d0b0-6b82eed03164 Address:172.27.0.7:8300} {Suffrage:Voter ID:d5aea96c-4e2b-86ca-5561-3ea97830e410 Address:172.27.0.4:8300} {Suffrage:Voter ID:eb4713cb-4061-8d85-d9c2-a8d907d56f85 Address:172.27.0.2:8300}]
	latest_configuration_index = 42
	num_peers = 2
	protocol_version = 3
	protocol_version_max = 3
	protocol_version_min = 0
	snapshot_version_max = 1
	snapshot_version_min = 0
	state = Follower
	term = 3
runtime:
	arch = amd64
	cpu_count = 4
	goroutines = 90
	max_procs = 4
	os = linux
	version = go1.12.8
serf_lan:
	coordinate_resets = 0
	encrypted = false
	event_queue = 0
	event_time = 3
	failed = 0
	health_score = 0
	intent_queue = 0
	left = 0
	member_time = 5
	members = 6
	query_queue = 0
	query_time = 1
serf_wan:
	coordinate_resets = 0
	encrypted = false
	event_queue = 0
	event_time = 1
	failed = 0
	health_score = 0
	intent_queue = 0
	left = 0
	member_time = 7
	members = 3
	query_queue = 0
	query_time = 1
```
</details>

このコマンドでは`Agent`, `Raft`, `Serf`などの情報を概要レベルで確認することが出来ます。`Raft`はデータのレプリケーションや leader election を行うためのプロトコルです。`Serf`は Gossip プロトコルの内部で利用されているライブラリで、Consul の状態をクラスタで共有するための仕組みです。

## members

次は Consul のクラスタやクラアイントのリストを表示させるコマンドです。

```console
$ consul members
Node          Address          Status  Type    Build  Protocol  DC   Segment
82963a2d4324  172.27.0.2:8301  alive   server  1.6.0  2         dc1  <all>
8a56d570dc47  172.27.0.7:8301  alive   server  1.6.0  2         dc1  <all>
c766fc349e3c  172.27.0.4:8301  alive   server  1.6.0  2         dc1  <all>
2bbc5385226e  172.27.0.3:8301  alive   client  1.6.0  2         dc1  <default>
b51ccbf4f1a3  172.27.0.6:8301  alive   client  1.6.0  2         dc1  <default>
b6075d14f88b  172.27.0.5:8301  alive   client  1.6.0  2         dc1  <default>
```

ここでは Server ノードとエージェントが入っているクライアントも含まれています。Status は`alive`, `left`, `failed`のいずれかがあります。一つコンテナを停止してみましょう。`Node`がコンテナの ID になっています。いずれかをメモしましょう。

```shell
$ docker stop 82963a2d4324
```

```console
$ consul members
Node          Address          Status  Type    Build  Protocol  DC   Segment
82963a2d4324  172.27.0.2:8301  failed   server  1.6.0  2         dc1  <all>
8a56d570dc47  172.27.0.7:8301  alive   server  1.6.0  2         dc1  <all>
c766fc349e3c  172.27.0.4:8301  alive   server  1.6.0  2         dc1  <all>
2bbc5385226e  172.27.0.3:8301  alive   client  1.6.0  2         dc1  <default>
b51ccbf4f1a3  172.27.0.6:8301  alive   client  1.6.0  2         dc1  <default>
b6075d14f88b  172.27.0.5:8301  alive   client  1.6.0  2         dc1  <default>
```

`failed`に変化したことがわかるでしょう。しばらくすると Autopilot 機能により自動的に`left`となりクラスタから切り離されます。Autopilot 機能はこのあともう少し見ていきます。


`-detailed`オプションをつけることで詳細な情報を見ることが出来ます。一旦`Ctr+C`で全コンテナを停止して再起動しましょう。

```shell
$ docker-compose down
$ docker-compose up
```

## catalog

catalog コマンドは Consul のサービスカタログにアクセスするためのコマンドです。

```console
$ consul catalog datacenters
dc1
```

ノード一覧を見るためには`nodes`オプションをつけます。

```console
$ consul catalog nodes
Node          ID        Address     DC
24c806eb93c7  e70f6d40  172.28.0.2  dc1
268b3f573711  4f309842  172.28.0.6  dc1
4249d6f476cc  bd27f783  172.28.0.4  dc1
6eed7e05c966  1a358267  172.28.0.7  dc1
76d8c6bcb5d3  78639829  172.28.0.5  dc1
e52211884a6b  2bc3a125  172.28.0.3  dc1
```

サービス一覧は`services`です。

```console
$ consul catalog services
consul
```

一つサービスを追加してみましょう。

```shell
consul services register \
-name=dummy \
-id=dummy \
-address=127.0.0.1 \
-port=4321 \
-tag=thisisdummy
```

```console
$ consul catalog services
consul
dummy
```

サービスカタログが更新されているはずです。またサービス名をキーにフィルタリングなどもできます。

```console
$ consul catalog nodes -service=dummy
Node          ID        Address     DC                                                                                        
4249d6f476cc  bd27f783  172.28.0.4  dc1 
```

## monitor

monitor コマンドは直近のログを出力してストリームするためのコマンドです。

```console
$ consul monitor
2019/11/24 04:54:42 [INFO]  raft: Initial configuration (index=0): []
2019/11/24 04:54:42 [INFO]  raft: Node at 172.28.0.4:8300 [Follower] entering Follower state (Leader: "")
2019/11/24 04:54:42 [INFO] serf: EventMemberJoin: 4249d6f476cc.dc1 172.28.0.4
2019/11/24 04:54:42 [INFO] serf: EventMemberJoin: 4249d6f476cc 172.28.0.4
2019/11/24 04:54:42 [INFO] agent: Started DNS server 0.0.0.0:8600 (udp)
2019/11/24 04:54:42 [INFO] consul: Adding LAN server 4249d6f476cc (Addr: tcp/172.28.0.4:8300) (DC: dc1)
```

また`-log-level`を変更することもできます。

```console
$ consul monitor -log-level=debug
```

デバッグログが表示されるでしょう。

## rtt

`rtt`は Round Trip Time の略です。Consul のノード間の通信の RTT を推定するためのツールです。まずはノードの ID を取得しましょう。任意の 2 つの ID をメモしてコマンドの引数にセットします。

```console
$ consul rtt 82963a2d4324 8a56d570dc47
Estimated 6eed7e05c966 <-> 268b3f573711 rtt: 0.989 ms (using LAN coordinates)
```

Consul では`network tomograpyh`という仕組みを使って計算し、Serf を通してクラスタに伝達されます。詳細は[こちら](https://www.consul.io/docs/internals/coordinates.html)を見てください。

## operator

operator コマンドは raft などの Consul のサブシステムを扱うための運用者向けのコマンドです。

まずは`raft`を扱ってみます。`list-peers`で現在のクラスタ内の状況を確認できます。

```console
$ consul operator raft list-peers
Node          ID                                    Address          State     Voter  RaftProtocol
4249d6f476cc  bd27f783-1d3d-8fae-acf5-5de668d7581f  172.28.0.4:8300  leader    true   3
76d8c6bcb5d3  78639829-bcab-be66-f7e8-8a1c59062a67  172.28.0.5:8300  follower  true   3
6eed7e05c966  1a358267-3d7a-3288-2ba1-db1a21a16bc4  172.28.0.7:8300  follower  true   3
```

`State`はクラスタ内の役割、`Voter`はそのノードが Leader Election のメンバーかどうかを表示しています。OSS だと全て True になります。興味がある方は[Enterprise 版機能の紹介](https://docs.google.com/presentation/d/1EdCRjc9nCBf9txf4xk__8BOUFYr5WhObsjz4IliAMgg/edit?usp=sharing)を見てください。

次は`autopilot`です。Autopilot は意図せず停止してしまったサーバの clean up やクラスタの状態のモニタリングや安定したサーバの情報を提供するなどを行います。

この機能に対して様々設定を行いますが operator コマンドではその設定の更新や確認を行うことが出来ます。

```console
$ consul operator autopilot get-config
CleanupDeadServers = true
LastContactThreshold = 200ms
MaxTrailingLogs = 250
ServerStabilizationTime = 10s
RedundancyZoneTag = ""
DisableUpgradeMigration = false
UpgradeVersionTag = ""
```

`CleanupDeadServers`は`failed`になっているサーバを自動でクラスタから切り離す機能です。デフォルトだと`force-leave`というコマンドで明示的に切り離すか 72 時間後に切り離されるまで待たないといけません。

この設定を`false`にして無効にします。

```shell
$ consul operator autopilot set-config -cleanup-dead-servers=false
```

`Type`が`server`の Node を一つ停止してみましょう。

```console
$ consul members
Node          Address          Status  Type    Build  Protocol  DC   Segment
4249d6f476cc  172.28.0.4:8301  alive   server  1.6.0  2         dc1  <all>
6eed7e05c966  172.28.0.7:8301  alive   server  1.6.0  2         dc1  <all>
76d8c6bcb5d3  172.28.0.5:8301  alive   server  1.6.0  2         dc1  <all>
24c806eb93c7  172.28.0.2:8301  alive   client  1.6.0  2         dc1  <default>
268b3f573711  172.28.0.6:8301  alive   client  1.6.0  2         dc1  <default>
e52211884a6b  172.28.0.3:8301  alive   client  1.6.0  2         dc1  <default>
```

停止します。

```shell
$ docker stop 76d8c6bcb5d3
```

`failed`になっているでしょう。

```console
$ consul members
Node          Address          Status  Type    Build  Protocol  DC   Segment
4249d6f476cc  172.28.0.4:8301  failed   server  1.6.0  2         dc1  <all>
6eed7e05c966  172.28.0.7:8301  alive   server  1.6.0  2         dc1  <all>
76d8c6bcb5d3  172.28.0.5:8301  alive   server  1.6.0  2         dc1  <all>
24c806eb93c7  172.28.0.2:8301  alive   client  1.6.0  2         dc1  <default>
268b3f573711  172.28.0.6:8301  alive   client  1.6.0  2         dc1  <default>
e52211884a6b  172.28.0.3:8301  alive   client  1.6.0  2         dc1  <default>
```

`members`の章で試した時とは違い自動で切り離されません。この状態だとクラスタ通信が発生し続けるため明示的に切り離さないといけません。

```shell
$ consul force-leave 76d8c6bcb5d3
```

`left`に変更されたはずです。話が少しそれましたが`consul operator autopilot`ではこのように Autopilot 機能の設定変更や確認を行うことが出来ます。

オートパイロットについては[こちら](https://learn.hashicorp.com/consul/day-2-operations/autopilot)を参考にしてください。

## snapshot

最後はバックアップを取得するためのコマンドです。また Restore も出来ます。

まずはバックアップの取得です。

```shell
$ consul catalog services
consul
dummy

$ consul snapshot save test.snap
```

inspect でスナップショットの情報を見ることが出来ます。

```shell
$ consul snapshot inspect test.snap
```

ここで一旦先ほど登録した`dummy`をカタログから外しておきます。

```console
$ consul services deregister -id=dummy

$ consul catalog services
consul
```

restore コマンドでリストアします。

```shell
$ consul snapshot restore test.snap
```

再度サービスカタログを見てみましょう。

```console
$ consul catalog services
consul
dummy
```

リストアされました。Enterprise 版では agent コマンドを使ってデーモンプロセスとして定期的なバックアップ取得の自動化やバックアップ保存先の指定などの細かい設定を行うことが出来ます。

興味がある方は[Enterprise 版機能の紹介](https://docs.google.com/presentation/d/1EdCRjc9nCBf9txf4xk__8BOUFYr5WhObsjz4IliAMgg/edit?usp=sharing)を見てください。

最後に`Ctr+C`で抜けて全コンテナを停止しておきましょう。

```shell
$ docker-compose down
```

## 参考リンク
* [Consul CLI](https://www.consul.io/docs/commands/index.html)