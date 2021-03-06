# Consul clusterの構築　Vagrant版

## 前提条件

* [Vagrant](https://www.vagrantup.com/downloads.html)
* [Virtualbox](https://www.virtualbox.org/wiki/Downloads)
* [Vagrantfile](Vagrantfile)

## Vagrantの実行

Vagrantfileが置いてあるディレクトリに移動し、以下のコマンドを打ちます。
```shell
vagrant up
```

もしVagrant実行で下記のエラーが発生した場合は、Vagrantのvbguestプラグインが古い可能性があります。以下のコマンドでアップデートしてください。

```shell
vagrant plugin update vagrant-vbguest
```

Vagrantが自動的にVMやConsulのバイナリをダウンロードしてクラスタが構築されます。

この設定では、１台のConsulサーバーと２台のConsulクライアントが構築されます。

* Server
  - IP address: 172.20.20.3
  - Hostname: server1
  - Vagrantからのログイン方法：
    - `vagrant ssh s1`

* Client1
  - IP address: 172.20.20.4
  - Hostname: Client1
  - Vagrantからのログイン方法：
    - `vagrant ssh c1`

* Client2
  - IP address: 172.20.20.5
  - Hostname: Client1
  - Vagrantからのログイン方法：
    - `vagrant ssh c2`

## 動作確認

どれでも好きなVMにログインして、以下のコマンドを打ってください。

```shell
consul members
```

このコマンドは現在参加していくクラスタにJoinしている全てのメンバーを表示します。以下のような出力であれば無事にクラスタが構築されています。

```console
vagrant@client2:~$ consul members
Node     Address           Status  Type    Build  Protocol  DC   Segment
server1  172.20.20.3:8301  alive   server  1.5.3  2         dc1  <all>
client1  172.20.20.4:8301  alive   client  1.5.3  2         dc1  <default>
client2  172.20.20.5:8301  alive   client  1.5.3  2         dc1  <default>
```
