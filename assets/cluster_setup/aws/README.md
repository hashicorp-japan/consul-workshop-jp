# AWS 上へ Consul クラスタを構築する

Consul クラスタの構築方法は多くの方法がありますが、ここでは最も汎用な Module を使った構築方法を行います。


## 必要なコードをダウンロード

まずは以下の Github Repository を Clone もしくは Zip をダウンロードしてください。

[https://github.com/hashicorp/terraform-aws-consul](https://github.com/hashicorp/terraform-aws-consul)

ちなみにこの Repository は Terraform の[Public Module Registry](https://registry.terraform.io/modules/hashicorp/consul/aws/0.7.3)上で公開されている Module の Repository でもあります。非常に汎用的に作られているので、すでに Terraform でインフラ構築をしてるなら、この Module を組み込むのもいいかもしれません。

Clone もしくはダウンロードができましたら、いくつかの修正を加える必要があります。


## 変数を修正

variables.tf (top directory にあるもの）を開いてくさあい。

* AMI の Image ID
	* お使いの Region により以下のリストより選んでください。
		[https://github.com/hashicorp/terraform-aws-consul/blob/master/_docs/ubuntu16-ami-list.md](https://github.com/hashicorp/terraform-aws-consul/blob/master/_docs/ubuntu16-ami-list.md)
	* ちなみにここではあらかじめ準備してある汎用の AMI を使いますが、もし皆さんの環境で Consul クラスタを構築する際は、かならず独自に AMI を準備してください。AMI の構築には、弊社の製品である(Packer)[https://www.packer.io/]をお勧めします。

```hcl
variable "ami_id" {
  description = "The ID of the AMI to run in the cluster. This should be an AMI built from the Packer template under examples/consul-ami/consul.json. To keep this example simple, we run the same AMI on both server and client nodes, but in real-world usage, your client nodes would also run your apps. If the default value is used, Terraform will look up the latest AMI build automatically."
  type        = string
  default     = "ami-016727cd7a1a22e9a"    # リストより選んでください
}
```


次に、ハンズオン用としてミニマムなクラスタサイズにします。ここではサーバー数を１とし、クライアントを２とします。（お好きな数を Provision しても構いません）

```hcl
variable "num_servers" {
  description = "The number of Consul server nodes to deploy. We strongly recommend using 3 or 5."
  type        = number
  default     = 1
1

variable "num_clients" {
  description = "The number of Consul client nodes to deploy. You typically run the Consul client alongside your apps, so set this value to however many Instances make sense for your app code."
  type        = number
  default     = 2
}
```

また必須ではないですが、もし Provision したサーバーに SSH でアクセスしたいのであれば、ssh keypair も指定してください。

```hcl
variable "ssh_key_name" {
  description = "The name of an EC2 Key Pair that can be used to SSH to the EC2 Instances in this cluster. Set to an empty string to not associate a Key Pair."
  type        = string
  default     = "your-key-pair" 
}
```

## Provision

それでは Provision してみましょう。

まずは、AWS にアクセスするためにクレデンシャルを環境変数に設定してください。

```shell
export AWS_ACCESS_KEY_ID="xxxxxxxxxxxxxxx"
export AWS_SECRET_ACCESS_KEY="xxxxxxxxxxxxxxxxx"
export AWS_DEFAULT_REGION="ap-northeast-1"    # 日本
```

次に Terraform を初期化し Provision に備えます。

```shell
terraform init
```

問題なく Plug-in などがダウンロードできたら、Plan を行います。

```shell
terraform plan
```

問題なければ実際に Provision します。


```shell
terraform apply -auto-approve
```

これで Consul クラスタが AWS 上に構築されます。


## Consul クラスタへのアクセス確認

AWS の Console、CLI もしくは、以下のスクリプトを実行して Consul サーバーの IP アドレスを取得します。

```console
$examples/consul-examples-helper/consul-examples-helper.sh
2019-08-21 15:46:37 [INFO] [consul-examples-helper.sh] Looking up public IP addresses for 1 Consul server EC2 Instances.
2019-08-21 15:46:37 [INFO] [consul-examples-helper.sh] Fetching public IP addresses for EC2 Instances in ap-northeast-1 with tag consul-servers=masa-consul-example
2019-08-21 15:46:38 [INFO] [consul-examples-helper.sh] Found all 1 public IP addresses!
2019-08-21 15:46:38 [INFO] [consul-examples-helper.sh] Waiting for 1 Consul servers to register in the cluster
2019-08-21 15:46:38 [INFO] [consul-examples-helper.sh] Running 'consul members' command against server at IP address 13.115.58.242
2019-08-21 15:46:38 [INFO] [consul-examples-helper.sh] All 1 Consul servers have registered in the cluster!

Your Consul servers are running at the following IP addresses:

    13.115.58.242

Some commands for you to try:

    consul members -http-addr=13.115.58.242:8500
    consul kv put -http-addr=13.115.58.242:8500 foo bar
    consul kv get -http-addr=13.115.58.242:8500 foo
    consul kv get -http-addr=:8500 foo
    consul kv get -http-addr=:8500 foo

To see the Consul UI, open the following URL in your web browser:

    http://13.115.58.242:8500/ui/

```

ここで得られた IP アドレスを環境変数に追加します。

```shell
$export CONSUL_HTTP_ADDR=13.115.58.242:8500
```

これにより、このシェル上での`consul`コマンドは API リクエストを AWS 上の Consul サーバーへ送信するようになります。

試してみましょう。

```shell
$consul members
Node                 Address             Status  Type    Build  Protocol  DC              Segment
i-0926908f7046fd232  172.31.15.202:8301  alive   server  1.5.3  2         ap-northeast-1  <all>
i-007526692a3df2f85  172.31.31.194:8301  alive   client  1.5.3  2         ap-northeast-1  <default>
i-0cc27869de397b707  172.31.9.220:8301   alive   client  1.5.3  2         ap-northeast-1  <default>
```

このようにメンバーが表示されれば準備完了です。


