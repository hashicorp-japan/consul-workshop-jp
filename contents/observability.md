# Consul のメトリクスを監視する

Consul では Consul サーバやその配下にあるサービスやノードなどのメトリクスを効率的に可視化するための様々な機能が用意されています。

* Telemetry
* L7 Obervalibity
* Traffic Tracing

などです。ここでは`Telemetry`, `L7`を試してみます。Tracing については後で追加します。

[Intentions の章](https://github.com/hashicorp-japan/consul-workshop/blob/master/contents/intentions.md)で利用したアプリを使うのでこの手順が終わっていない方は`アプリのデプロイ`までを終了させてください。

終了している方は以下の手順を進めてください。また終了している方も手順に沿って起動を行なってください。

```console
$ pwd
/path/to/consul-workshop/consul-intentions-demo
```

## Telemetry の設定を行う

まずは各サーバにインストールされている Consul Agent を使ってメトリクスを取得するパターンを試してみます。Consul では `statsite`, `statsd`にログを転送したり、`Prometheus`のフォーマットでログを出力させ、スクレイピングさせたり出来ます。

ここでは Prometheus を使ってみます。

`consul_config/consul-config.hcl`ファイルの最後に以下の行を追加します。

```hcl
telemetry = {
  "prometheus_retention_time" = "3h"
}
```

この設定を加えることで Prometheus のフォーマットでメトリクスが expose されます。また、ここではメトリクスの保持時間として 3 時間を指定しています。

設定はこれだけです。設定を反映させるために Consul を再起動します。

```shell
$ docker-compose down
$ docker-compose up -d
```

`http://127.0.0.1:8500/v1/agent/metrics?format=prometheus`にアクセスすると多くのメトリクスが出力されるでしょう。

<details><summary>出力例</summary>

```
# HELP consul_9ad423be44a4_autopilot_failure_tolerance consul_9ad423be44a4_autopilot_failure_tolerance
# TYPE consul_9ad423be44a4_autopilot_failure_tolerance gauge
consul_9ad423be44a4_autopilot_failure_tolerance 0
# HELP consul_9ad423be44a4_autopilot_healthy consul_9ad423be44a4_autopilot_healthy
# TYPE consul_9ad423be44a4_autopilot_healthy gauge
consul_9ad423be44a4_autopilot_healthy 1
# HELP consul_9ad423be44a4_consul_cache_entries_count consul_9ad423be44a4_consul_cache_entries_count
# TYPE consul_9ad423be44a4_consul_cache_entries_count gauge
consul_9ad423be44a4_consul_cache_entries_count 22
# HELP consul_9ad423be44a4_raft_commitNumLogs consul_9ad423be44a4_raft_commitNumLogs
# TYPE consul_9ad423be44a4_raft_commitNumLogs gauge
consul_9ad423be44a4_raft_commitNumLogs 1
# HELP consul_9ad423be44a4_raft_leader_dispatchNumLogs consul_9ad423be44a4_raft_leader_dispatchNumLogs
# TYPE consul_9ad423be44a4_raft_leader_dispatchNumLogs gauge
consul_9ad423be44a4_raft_leader_dispatchNumLogs 1
# HELP consul_9ad423be44a4_runtime_alloc_bytes consul_9ad423be44a4_runtime_alloc_bytes
# TYPE consul_9ad423be44a4_runtime_alloc_bytes gauge
consul_9ad423be44a4_runtime_alloc_bytes 1.4908912e+07
# HELP consul_9ad423be44a4_runtime_free_count consul_9ad423be44a4_runtime_free_count
# TYPE consul_9ad423be44a4_runtime_free_count gauge
consul_9ad423be44a4_runtime_free_count 4.054047e+06
# HELP consul_9ad423be44a4_runtime_heap_objects consul_9ad423be44a4_runtime_heap_objects
# TYPE consul_9ad423be44a4_runtime_heap_objects gauge
consul_9ad423be44a4_runtime_heap_objects 54955
# HELP consul_9ad423be44a4_runtime_malloc_count consul_9ad423be44a4_runtime_malloc_count
# TYPE consul_9ad423be44a4_runtime_malloc_count gauge
consul_9ad423be44a4_runtime_malloc_count 4.109002e+06
```
</details>

各メトリクスの情報は[こちら](https://www.consul.io/docs/agent/telemetry.html)を参考にして下さい。

## L7 Observability の設定を行う

Consul では上記で設定した Consul クラスタに関わるメトリクスの他に Envoy と連携をして各サービスの L7 を含めたメトリクスを簡単に出力することが出来ます。

`consul_config/consul-config.hcl`ファイルの最後に以下の行を追加します。

```hcl
config_entries {
  bootstrap =
  [{
    kind = "proxy-defaults"
    name = "global"
    config {
      "envoy_prometheus_bind_addr" = "0.0.0.0:9102"
    }
  }]
}
```

各サービスに関わるサイドカーの設定を`proxy-defaults`としてセットしています。各サービスの Envoy で出力されるメトリクスを`9102`でスクレイプ出来る様にしてあります。

設定を反映させるために Consul を再起動します。

```shell
$ docker-compose down
$ docker-compose up -d
```

この設定を行うと全てのサイドカーに反映されますが、一つのアプリからどの様なメトリクスが出力されるかを見てみましょう。

`docker-compose.yml`の`hashicorpjapanapp`の設定に`9102:9102`でポートフォワードする様に設定されています。

```console
$ curl http://127.0.0.1:9102/metrics

# TYPE envoy_ext_authz_connect_authz_failure_mode_allowed counter
envoy_ext_authz_connect_authz_failure_mode_allowed{local_cluster="hashicorpjapanapp"} 0
# TYPE envoy_ext_authz_connect_authz_error counter
envoy_ext_authz_connect_authz_error{local_cluster="hashicorpjapanapp"} 0
# TYPE envoy_ext_authz_connect_authz_denied counter
envoy_ext_authz_connect_authz_denied{local_cluster="hashicorpjapanapp"} 0
# TYPE envoy_ext_authz_connect_authz_total counter
envoy_ext_authz_connect_authz_total{local_cluster="hashicorpjapanapp"} 0
# TYPE envoy_ext_authz_connect_authz_ok counter
envoy_ext_authz_connect_authz_ok{local_cluster="hashicorpjapanapp"} 0
# TYPE envoy_ext_authz_connect_authz_cx_closed counter
envoy_ext_authz_connect_authz_cx_closed{local_cluster="hashicorpjapanapp"} 0

~~~~
```

メトリクスが出力されています。

次にこれらのメトリクスを Prometheus で収集してみましょう。

## Prometheus で収集する

Prometheus の設定ファイルを作ります。Prometheus では`consul_sd_config`という Consul の Service Dicovery を利用して、Consul 配下にあるサービス群を動的に監視出来るような連携機能が用意されています。

```shell
$ cat << EOF > prometheus-envoy-intensions-demo.yml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
  external_labels:
    origin_prometheus: prometheus01
# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

scrape_configs:
  - job_name: 'consul-metrics'
    metrics_path: /v1/agent/metrics
    scheme: http
    params:
      format: ['prometheus']
    scrape_interval: 10s
    scrape_timeout: 5s  
    consul_sd_configs:
      - server: '10.5.0.2:8500'
        services:    
          - consul
    relabel_configs:
      - source_labels: [__address__]
        separator:     ':'
        regex:         '(.*):(.*)'
        target_label:  '__address__'
        replacement:   '\${1}:8500'
  - job_name: 'envoy-metrics'
    scrape_interval: 10s
    scrape_timeout: 5s  
    consul_sd_configs:
      - server: '10.5.0.2:8500'
        services:    
          - hashiapp
          - corpapp
          - japanapp
          - hashicorpjapanapp
    relabel_configs:
      - source_labels: [__address__]
        separator:     ':'
        regex:         '(.*):(.*)'
        target_label:  '__address__'
        replacement:   '\${1}:9102'
EOF
```

次に Docker Compose に Prometheus を追加します。

```shell
cat << EOF > docker-compose.yml
version: "3.3"
services:

  consul:
    image: consul:1.6.0
    command: ["consul","agent","-config-file=/config/consul-config.hcl","-config-dir=/config"]
    volumes:
      - "./consul_config:/config"
      - "./consul_data:/data"
    ports:
      - 8500:8500
    networks:
      vpcbr:
        ipv4_address: 10.5.0.2

  hashicorpjapanapp:
    build:
      context: ./hashicorpjapanapp
      dockerfile: Dockerfile
    networks:
      vpcbr:
        ipv4_address: 10.5.0.3
    ports:
      - 8080:8080
      - 9102:9102
  hashicorpjapanapp_envoy:
    image: nicholasjackson/consul-envoy:v1.6.0-v0.10.0
    environment:
      CONSUL_HTTP_ADDR: 10.5.0.2:8500
      CONSUL_GRPC_ADDR: 10.5.0.2:8502
      SERVICE_CONFIG: /config/sidecar-hashicorpjapanapp.json
    volumes:
      - "./consul.d:/config"
    command: ["consul", "connect", "envoy","-sidecar-for", "hashicorpjapanapp"]
    network_mode: "service:hashicorpjapanapp"
 
  hashiapp:
    build:
      context: ./hashiapp
      dockerfile: Dockerfile
    networks:
      vpcbr:
        ipv4_address: 10.5.0.4
    ports:
      - 1010:8080
  hashiapp_envoy:
    image: nicholasjackson/consul-envoy:v1.6.0-v0.10.0
    environment:
      CONSUL_HTTP_ADDR: 10.5.0.2:8500
      CONSUL_GRPC_ADDR: 10.5.0.2:8502
      SERVICE_CONFIG: /config/sidecar-hashiapp.json
    volumes:
      - "./consul.d:/config"
    command: ["consul", "connect", "envoy","-sidecar-for", "hashiapp"]
    network_mode: "service:hashiapp"
  
  corpapp:
    build:
      context: ./corpapp
      dockerfile: Dockerfile
    networks:
      vpcbr:
        ipv4_address: 10.5.0.5
    ports:
      - 2020:8080
  corpapp_envoy:
    image: nicholasjackson/consul-envoy:v1.6.0-v0.10.0
    environment:
      CONSUL_HTTP_ADDR: 10.5.0.2:8500
      CONSUL_GRPC_ADDR: 10.5.0.2:8502
      SERVICE_CONFIG: /config/sidecar-corpapp.json
    volumes:
      - "./consul.d:/config"
    command: ["consul", "connect", "envoy","-sidecar-for", "corpapp"]
    network_mode: "service:corpapp"

  japanapp:
    build:
      context: ./japanapp
      dockerfile: Dockerfile
    networks:
      vpcbr:
        ipv4_address: 10.5.0.6
    ports:
      - 3030:8080
  japanapp_envoy:
    image: nicholasjackson/consul-envoy:v1.6.0-v0.10.0
    environment:
      CONSUL_HTTP_ADDR: 10.5.0.2:8500
      CONSUL_GRPC_ADDR: 10.5.0.2:8502
      SERVICE_CONFIG: /config/sidecar-japanapp.json
    volumes:
      - "./consul.d:/config"
    command: ["consul", "connect", "envoy","-sidecar-for", "japanapp"]
    network_mode: "service:japanapp"

  unintentionalapp:
    build:
      context: ./hashicorpjapanapp
      dockerfile: Dockerfile
    networks:
      vpcbr:
        ipv4_address: 10.5.0.7
    ports:
      - 9090:8080
  unintentionalapp_envoy:
    image: nicholasjackson/consul-envoy:v1.6.0-v0.10.0
    environment:
      CONSUL_HTTP_ADDR: 10.5.0.2:8500
      CONSUL_GRPC_ADDR: 10.5.0.2:8502
      SERVICE_CONFIG: /config/sidecar-unintentionalapp.json
    volumes:
      - "./consul.d:/config"
    command: ["consul", "connect", "envoy","-sidecar-for", "unintentionalapp"]
    network_mode: "service:unintentionalapp"
    
  prometheus-server:
    image: prom/prometheus
    ports:
      - 9999:9090
    volumes:
      - ./prometheus-envoy-intensions-demo.yml:/etc/prometheus/prometheus.yml
    networks:
      vpcbr:
        ipv4_address: 10.5.0.9
  
networks:
  vpcbr:
    driver: bridge
    ipam:
     config:
       - subnet: 10.5.0.0/16
EOF
```

一旦 Docker を再起動しましょう。

```shell
$ docker-compose down
$ docker-compose up -d
```

起動後、`http://localhost:9999/graph`にアクセスをすると Prometheus の画面が見れるでしょう。

検索ボックスから任意のメトリクスを入力してグラフを作ってみてください。

この様に Prometheus でメトリクスを集約し、Grafana などのダッシュボードのツールで環境の一括監視を簡単に行うことが出来ます。

## 参考リンク
* [Telemetry](https://www.consul.io/docs/agent/telemetry.html)
* [Telemetry Configuration](https://www.consul.io/docs/agent/options.html#telemetry)
* [Observablity](https://www.consul.io/docs/connect/observability.html)
* [Envoy Integration](https://www.consul.io/docs/connect/proxies/envoy.html#bootstrap-configuration)
* [Prometheus consul_sd_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#consul_sd_config)