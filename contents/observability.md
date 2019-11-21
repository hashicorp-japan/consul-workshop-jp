# Consulのメトリクスを監視する

ConsulではConsulサーバやその配下にあるサービスやノードなどのメトリクスを効率的に可視化するための様々な機能が用意されています。

* Telemetry
* L7 Obervalibity
* Traffic Tracing

などです。ここでは`Telemetry`, `L7`を試してみます。Tracingについては後で追加します。

[Intentionsの章](https://github.com/hashicorp-japan/consul-workshop/blob/master/contents/intentions.md)で利用したアプリを使うのでこの手順が終わっていない方は`アプリのデプロイ`までを終了させてください。

終了している方は以下の手順を進めてください。また終了している方も手順に沿って起動を行なってください。

## Telemetryの設定を行う

まずは各サーバにインストールされているConsul Agentを使ってメトリクスを取得するパターンを試してみます。Consulでは `statsite`, `statsd`にログを転送したり、`Prometheus`のフォーマットでログを出力させ、スクレイピングさせたり出来ます。

ここではPrometheusを使ってみます。

`consul_config/consul-config.hcl`ファイルの最後に以下の行を追加します。

```hcl
telemetry = {
  "prometheus_retention_time" = "3h",
}
```

この設定を加えることでPrometheusのフォーマットでメトリクスがexposeされます。また、ここではメトリクスの保持時間として3時間を指定しています。

設定はこれだけです。設定を反映させるために`consul reload`コマンドを実行してください。

```shell
$ consul reload
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

## L7 Observabilityの設定を行う

Consulでは上記で設定したConsulクラスタに関わるメトリクスの他にEnvoyと連携をして各サービスのL7を含めたメトリクスを簡単に出力することが出来ます。

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

各サービスに関わるサイドカーの設定を`proxy-defaults`としてセットしています。各サービスのEnvoyで出力されるメトリクスを`9102`でスクレイプ出来る様にしてあります。

この設定を行うと全てのサイドカーに反映されますが、一つのアプリからどの様なメトリクスが出力されるかを見てみましょう。

`docker-compose.yml`の`hashicorpjapanapp`の設定に`9102:9102`でポートフォワードする様に設定されています。

```consul
$ http://127.0.0.1:9102/metrics

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

次にこれらのメトリクスをPrometheusで収集してみましょう。

## Prometheusで収集する

Prometheusの設定ファイルを作ります。Prometheusでは`consul_sd_config`というConsulのService Dicoveryを利用して、Consul配下にあるサービス群を動的に監視出来るような連携機能が用意されています。

```shell
$ cd path/to/consul-workshop
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
        replacement:   '${1}:8500'
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
        replacement:   '${1}:9102'
```

一旦Dockerを`Ctr+C`で停止をし、再起動しましょう。

```shell
$ docker-compose down
$ docker-compose up
```

起動後、`http://localhost:9999/graph`にアクセスをするとPrometheusの画面が見れるでしょう。

検索ボックスから任意のメトリクスを入力してグラフを作ってみてください。

この様にPrometheusでメトリクスを集約し、Grafanaなどのダッシュボードのツールで環境の一括監視を簡単に行うことが出来ます。

## 参考リンク
* [Telemetry](https://www.consul.io/docs/agent/telemetry.html)
* [Telemetry Configuration](https://www.consul.io/docs/agent/options.html#telemetry)
* [Observablity](https://www.consul.io/docs/connect/observability.html)
* [Envoy Integration](https://www.consul.io/docs/connect/proxies/envoy.html#bootstrap-configuration)
* [Prometheus consul_sd_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#consul_sd_config)