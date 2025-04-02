# HashiCorp Consul Workshop

[Consul](https://www.consul.io/) は HashiCorp が中心に開発をする OSS の Servcie Discovery や Service Mesh を実現するためのツールです。Service Discovery やヘルスチェックなどの基本的な機能に加えて mTLS, L7 Traffic Management やコンフィグレーション管理など様々な機能を提供しています。Consul はマルチプラットフォームでかつ全ての機能を HTTP API で提供しているため、環境やクライアントを問わず利用することができます。

本ワークショップは OSS の機能を中心に様々なユースケースに合わせたハンズオンを用意しています。

## Pre-requisite

* 環境
	* macOS or Linux

* ソフトウェア
	* Consul
	* Docker / Docker Compose
	* Java 12(いつか直します...)
	* jq, watch, wget, curl

## 資料

* [Consul Overview](https://docs.google.com/presentation/d/126Y5PgELCuYcR-j4IRQcj7sxczMKT0PgFWS8x9StHXE/edit?usp=sharing)

## Consul 概要の学習
こちらのビデオをご覧ください。

[HashiCorp Consul で始めるマルチクラウドサービスメッシュ](https://www.youtube.com/watch?v=QruAFCchmog)

## Agenda

1. [初めての Consul](contents/hello-consul.md)
1. [Service Discovery](contents/srd.md)
1. Service Mesh
	* [Sidecar Proxy の導入](contents/sidecar.md)
	* [Intensions](contents/intentions.md)
	* [L7 Traffic Management: Routing](contents/l7-routing.md)
	* [L7 Traffic Management: Splitting](contents/l7-splitting.md)
	* [Centerized Configration](contents/centerized-config.md)
	* [Consul Template](contents/consul-template.md)
	* [Observability](contents/observability.md)
	* Mesh Gateway
	* Certification Management
1. Kubernetes 連携
1. [運用系機能色々](contents/utilities.md)
1. [Enterprise 版機能の紹介](https://docs.google.com/presentation/d/1EdCRjc9nCBf9txf4xk__8BOUFYr5WhObsjz4IliAMgg/edit?usp=sharing)
