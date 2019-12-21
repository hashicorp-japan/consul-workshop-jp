# Hashicorp Consul Workshop

[Consul](https://www.consul.io/)はHashiCorpが中心に開発をするOSSのServcie DiscoveryやService Meshを実現するためのツールですService Discoveryやヘルスチェックなどの基本的な機能に加えてmTLS, L7 Traffic Managementやコンフィグレーション管理など様々な機能を提供しています。Consulはマルチプラットフォームでかつ全ての機能をHTTP APIで提供しているため、環境やクライアントを問わず利用することができます。

本ワークショップはOSSの機能を中心に様々なユースケースに合わせたハンズオンを用意しています。

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

## Agenda

1. [初めてのConsul](contents/hello-consul.md)
1. [consul cli](contents/cli.md)
1. [Service Discovery](contents/srd.md)
1. Service Mesh
	* [Sidecar Proxyの導入](contents/sidecar.md)
	* [Intensions](contents/intentions.md)
	* [L7 Traffic Management: Routing](contents/l7-routing.md)
	* [L7 Traffic Management: Splitting](contents/l7-splitting.md)
	* [Distributed Configration](contents/distributed-config.md)
	* [Consul Template](contents/scf.md)
	* [Observability](contents/observability.md)
	* Mesh Gateway
	* Certification Management
1. Kubernetes連携
1. [運用系機能色々](contents/utilities.md)
1. [Enterprise版機能の紹介](https://docs.google.com/presentation/d/1EdCRjc9nCBf9txf4xk__8BOUFYr5WhObsjz4IliAMgg/edit?usp=sharing)