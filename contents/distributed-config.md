# Consul Service Configuration

ConsulにはKey Valueのデータストアが内包されており、これを利用することで様々な処理を行うことができます。

* 各マイクロサービスへのコンフィグレーションの配布(Distributed Configurations)
* Consul管理は以下のサービスのコンフィグレーションの自動アップデート(Consul Template)
* KVSの値を監視して更新などをトリガーにしたイベント処理(Watch)

などです。

Watchは別の章で扱いますのでここでは`Distributed Configurations`を試してみたいと思います。


## Consul KVの使い方

まず、Consul KVを使って簡単なデータのput/getをしてみましょう。

```console 
$ consul kv put my-first-kv/consulis useful
$ consul kv get my-first-kv/consulis
useful
```

`consulis`というキーに`useful`という値を入れて取り出しました。とても簡単です。Consul内のデータはConsulクラスタ内のノード同士でレプリケーションし、ユーザは意識することなくデータの冗長性を保つことができます。

## Distributed Configurations

ここでは簡単なWebアプリを使ってConsulからWebアプリに設定を注入してみます。ConsulのKVSに設定を格納し、それを各アプリケーションが取得します。まずはアプリのレポジトリをクローンして、セットアップをします。

```shell
$ git clone https://github.com/tkaburagi/consul-config-spring
$ cd consul-config-spring
```

アプリケーションをビルドしてDockerコンテナとして起動します。Consulサーバがすでに起動している場合は　`Ctrl+C`で停止してください。

```shell
$ ./run.sh
```

`http://127.0.0.1:8500`にブラウザでアクセスし、起動していることを確認してください。

`web-front`と`web-backend`の二つのアプリが起動し、それぞれローカルから`8080`, `9090`のポートでアクセスできるはずです。

```console
$ curl localhost:9090
Hello HashiCorp!!!!
```

```console
$ curl localhost:8080/greetings
MMMMMMMMMMMBTMMMM#TMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMM
MMMMMMMMMM!  .MMMM#   ?WMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMM
MMMMMMM      .MMMM#    .MMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMM
MM9^       ..MMMMM#    .MM@  7HMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMHMHMMMMMMMMMMMMWMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMM
M       .+MMMMMMMM#    .MM@    (MMMMMM   JMMMMM#   dMMMMMMMMMMMMMMMMMMMMMMMM   JMMMMMMMMM}  (MMMMM        MMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMM
M    .MMMM#M .MMMM#    .MM@    (MMMMMM   JMMMMM#   dMMMMMMMMMMMMMMMMMMMMMMMM   JMMMMMMMMMMMM(MM!         (MMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMM
M    .MMM    .MMMM#    -MM@    (MMMMMM   JMMMMM#   dMMMMMMMMMWMMMM#MMMMMMTMM   J#MM77TMMMYMM4M#   MMMMMMMMMMMMM777MWMMMMMMTM#MM#MMWHMM77TMMM
M    .MMM    .MMMM=    .MM@    (MMMMMM   ?MMMMM@   dMN         MMF   .....MM          .MM}  (M@   MMMMMMMMM$   ...   4M#      d#     ..   WM
M    .MMM              .MM@    (MMMMMM             dMMMMMMMM   dM)  (MMMMMMM   .NMMb   MM}  (M@   MMMMMMMM#   MMMMb  .M#   .JMM#   MMMM]  .M
M    .MMM    .....,    -MM@    (MMMMMM   (NNNNNm   dMM#        dMb.    _7WMM   JMMM@   MM}  (M@   MMMMMMMM#   MMMM@   M#   MMMM#   MMMM]  .M
M    .MMM    .MMMM#    .MM@    (MMMMMM   JMMMMM#   dMF  ....   dMMMNNg,   MM   JMMM@   MM}  (M@   MMMMMMMM#   MMMMF   M#   MMMM#   MMMM]  .M
M    .MMM    .MMMM# ..MMMMF    (MMMMMM   JMMMMM#   dM)  JMM#   dM#WMMM#   MM   JMMM@   MM}  (MN.         4N   ?MM#^  .M#   MMMM#   WMM#'  .M
M    .MMM    .MMMMNMMMMMM      (MMMMMM   JMMMMM#   dMN         dM{       .MM   JMMM@   MM}  (MMN,        ,MN,       .MM#   MMMM#         .MM
MNa. .MMM    .MMMMM#M        .gMMMMMMMMMMMMMMMMMMMMMMMMMNMMMMMMMMMMMMNMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMNNMMMMMMMMMNNMMMMMMMMMMMMMM#   MMNNMMMMM
MMMMMNMMM    .MMMM#      ..MMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMM#   MMMMMMMMM
MMMMMMMMMJ,  .MMMM#   .JMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMM# ..MMMMMMMMM
MMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMM

Message from Backend = Hello HashiCorp!!!!
Current java version = 12.0.2+10
App index = 1
Backend host = localhost%
```

## Spring Bootの設定を変更する

アプリが起動出来たら設定を変更してみます。Spring Bootには`Actuator`という機能があり、アプリケーションのメトリクスをHTTPのエンドポイントでExposeしてくれます。デフォルトでは限られたメトリクスのみ取得することが出来、特定のメトリクスは非公開になっています。これを有効化するには各アプリケーションの設定を書き換えてビルドし直してデプロイする必要があります。

Consulで設定を管理し、それを配布するとどのような振る舞いになるかを確認してみましょう。`Actuator`とログレベルの設定を変更します。

まずはエンドポイントの確認をします。

```console
$ curl localhost:8080/actuator | jq

{
  "_links": {
    "self": {
      "href": "http://localhost:8080/actuator",
      "templated": false
    },
    "health": {
      "href": "http://localhost:8080/actuator/health",
      "templated": false
    },
    "health-component": {
      "href": "http://localhost:8080/actuator/health/{component}",
      "templated": true
    },
    "health-component-instance": {
      "href": "http://localhost:8080/actuator/health/{component}/{instance}",
      "templated": true
    },
    "info": {
      "href": "http://localhost:8080/actuator/info",
      "templated": false
    }
  }
}

$ curl localhost:9090/actuator | jq

{
  "_links": {
    "self": {
      "href": "http://localhost:9090/actuator",
      "templated": false
    },
    "health-component": {
      "href": "http://localhost:9090/actuator/health/{component}",
      "templated": true
    },
    "health": {
      "href": "http://localhost:9090/actuator/health",
      "templated": false
    },
    "health-component-instance": {
      "href": "http://localhost:9090/actuator/health/{component}/{instance}",
      "templated": true
    },
    "info": {
      "href": "http://localhost:9090/actuator/info",
      "templated": false
    }
  }
}
```

上記がデフォルトで有効化されているエンドポイントです。興味のある方はいくつかアクセスしてみて出力結果を確認してみてください。

また、Docker Composeの`web_1`, `api_1`のログを見てデバッグログは出力されていないことを確認しておいてください。

Consul KVに設定を格納します。


```console
$ consul kv put config/application/data @app_config.yml
$ consul kv get config/application/data

management:
  endpoints:
    web:
      exposure:
        include: "*"
logging:
  level:
    org.springframework.web: DEBUG
```

`app_config.yml`の中身はこちらです。

```console
$ cat app_config.yml
management:
  endpoints:
    web:
      exposure:
        include: “*”
logging:
  level:
    org.springframework.web: DEBUG
```

全てのActuatorのエンドポイントを有効化し、Webのログをデバッグモードにしています。Docker Composeの出力を`Ctrl+C`で停止し再起動します。

```shell
$ docker-compose down
$ docker-compose up
```

Acuatorのエンドポイントを再度確認してみましょう。

```shell
$ curl 127.0.0.1:8080/actuator | jq
```

<details><summary>出力例</summary>

```json
{
  "_links": {
    "self": {
      "href": "http://127.0.0.1:8080/actuator",
      "templated": false
    },
    "archaius": {
      "href": "http://127.0.0.1:8080/actuator/archaius",
      "templated": false
    },
    "auditevents": {
      "href": "http://127.0.0.1:8080/actuator/auditevents",
      "templated": false
    },
    "beans": {
      "href": "http://127.0.0.1:8080/actuator/beans",
      "templated": false
    },
    "caches": {
      "href": "http://127.0.0.1:8080/actuator/caches",
      "templated": false
    },
    "caches-cache": {
      "href": "http://127.0.0.1:8080/actuator/caches/{cache}",
      "templated": true
    },
    "health-component-instance": {
      "href": "http://127.0.0.1:8080/actuator/health/{component}/{instance}",
      "templated": true
    },
    "health": {
      "href": "http://127.0.0.1:8080/actuator/health",
      "templated": false
    },
    "health-component": {
      "href": "http://127.0.0.1:8080/actuator/health/{component}",
      "templated": true
    },
    "conditions": {
      "href": "http://127.0.0.1:8080/actuator/conditions",
      "templated": false
    },
    "configprops": {
      "href": "http://127.0.0.1:8080/actuator/configprops",
      "templated": false
    },
    "bus-env-destination": {
      "href": "http://127.0.0.1:8080/actuator/bus-env/{destination}",
      "templated": true
    },
    "bus-env": {
      "href": "http://127.0.0.1:8080/actuator/bus-env",
      "templated": false
    },
    "bus-refresh": {
      "href": "http://127.0.0.1:8080/actuator/bus-refresh",
      "templated": false
    },
    "bus-refresh-destination": {
      "href": "http://127.0.0.1:8080/actuator/bus-refresh/{destination}",
      "templated": true
    },
    "env": {
      "href": "http://127.0.0.1:8080/actuator/env",
      "templated": false
    },
    "env-toMatch": {
      "href": "http://127.0.0.1:8080/actuator/env/{toMatch}",
      "templated": true
    },
    "info": {
      "href": "http://127.0.0.1:8080/actuator/info",
      "templated": false
    },
    "integrationgraph": {
      "href": "http://127.0.0.1:8080/actuator/integrationgraph",
      "templated": false
    },
    "loggers": {
      "href": "http://127.0.0.1:8080/actuator/loggers",
      "templated": false
    },
    "loggers-name": {
      "href": "http://127.0.0.1:8080/actuator/loggers/{name}",
      "templated": true
    },
    "heapdump": {
      "href": "http://127.0.0.1:8080/actuator/heapdump",
      "templated": false
    },
    "threaddump": {
      "href": "http://127.0.0.1:8080/actuator/threaddump",
      "templated": false
    },
    "metrics": {
      "href": "http://127.0.0.1:8080/actuator/metrics",
      "templated": false
    },
    "metrics-requiredMetricName": {
      "href": "http://127.0.0.1:8080/actuator/metrics/{requiredMetricName}",
      "templated": true
    },
    "scheduledtasks": {
      "href": "http://127.0.0.1:8080/actuator/scheduledtasks",
      "templated": false
    },
    "httptrace": {
      "href": "http://127.0.0.1:8080/actuator/httptrace",
      "templated": false
    },
    "mappings": {
      "href": "http://127.0.0.1:8080/actuator/mappings",
      "templated": false
    },
    "refresh": {
      "href": "http://127.0.0.1:8080/actuator/refresh",
      "templated": false
    },
    "features": {
      "href": "http://127.0.0.1:8080/actuator/features",
      "templated": false
    },
    "service-registry": {
      "href": "http://127.0.0.1:8080/actuator/service-registry",
      "templated": false
    },
    "bindings-name": {
      "href": "http://127.0.0.1:8080/actuator/bindings/{name}",
      "templated": true
    },
    "bindings": {
      "href": "http://127.0.0.1:8080/actuator/bindings",
      "templated": false
    },
    "channels": {
      "href": "http://127.0.0.1:8080/actuator/channels",
      "templated": false
    },
    "consul": {
      "href": "http://127.0.0.1:8080/actuator/consul",
      "templated": false
    }
  }
}
```
</details>

```shell
$ curl 127.0.0.1:9090/actuator | jq
```

<details><summary>出力例</summary>

```json
{
  "_links": {
    "self": {
      "href": "http://127.0.0.1:9090/actuator",
      "templated": false
    },
    "archaius": {
      "href": "http://127.0.0.1:9090/actuator/archaius",
      "templated": false
    },
    "auditevents": {
      "href": "http://127.0.0.1:9090/actuator/auditevents",
      "templated": false
    },
    "beans": {
      "href": "http://127.0.0.1:9090/actuator/beans",
      "templated": false
    },
    "caches": {
      "href": "http://127.0.0.1:9090/actuator/caches",
      "templated": false
    },
    "caches-cache": {
      "href": "http://127.0.0.1:9090/actuator/caches/{cache}",
      "templated": true
    },
    "health": {
      "href": "http://127.0.0.1:9090/actuator/health",
      "templated": false
    },
    "health-component": {
      "href": "http://127.0.0.1:9090/actuator/health/{component}",
      "templated": true
    },
    "health-component-instance": {
      "href": "http://127.0.0.1:9090/actuator/health/{component}/{instance}",
      "templated": true
    },
    "conditions": {
      "href": "http://127.0.0.1:9090/actuator/conditions",
      "templated": false
    },
    "configprops": {
      "href": "http://127.0.0.1:9090/actuator/configprops",
      "templated": false
    },
    "bus-env-destination": {
      "href": "http://127.0.0.1:9090/actuator/bus-env/{destination}",
      "templated": true
    },
    "bus-env": {
      "href": "http://127.0.0.1:9090/actuator/bus-env",
      "templated": false
    },
    "bus-refresh-destination": {
      "href": "http://127.0.0.1:9090/actuator/bus-refresh/{destination}",
      "templated": true
    },
    "bus-refresh": {
      "href": "http://127.0.0.1:9090/actuator/bus-refresh",
      "templated": false
    },
    "env": {
      "href": "http://127.0.0.1:9090/actuator/env",
      "templated": false
    },
    "env-toMatch": {
      "href": "http://127.0.0.1:9090/actuator/env/{toMatch}",
      "templated": true
    },
    "info": {
      "href": "http://127.0.0.1:9090/actuator/info",
      "templated": false
    },
    "integrationgraph": {
      "href": "http://127.0.0.1:9090/actuator/integrationgraph",
      "templated": false
    },
    "loggers": {
      "href": "http://127.0.0.1:9090/actuator/loggers",
      "templated": false
    },
    "loggers-name": {
      "href": "http://127.0.0.1:9090/actuator/loggers/{name}",
      "templated": true
    },
    "heapdump": {
      "href": "http://127.0.0.1:9090/actuator/heapdump",
      "templated": false
    },
    "threaddump": {
      "href": "http://127.0.0.1:9090/actuator/threaddump",
      "templated": false
    },
    "metrics": {
      "href": "http://127.0.0.1:9090/actuator/metrics",
      "templated": false
    },
    "metrics-requiredMetricName": {
      "href": "http://127.0.0.1:9090/actuator/metrics/{requiredMetricName}",
      "templated": true
    },
    "scheduledtasks": {
      "href": "http://127.0.0.1:9090/actuator/scheduledtasks",
      "templated": false
    },
    "httptrace": {
      "href": "http://127.0.0.1:9090/actuator/httptrace",
      "templated": false
    },
    "mappings": {
      "href": "http://127.0.0.1:9090/actuator/mappings",
      "templated": false
    },
    "refresh": {
      "href": "http://127.0.0.1:9090/actuator/refresh",
      "templated": false
    },
    "features": {
      "href": "http://127.0.0.1:9090/actuator/features",
      "templated": false
    },
    "service-registry": {
      "href": "http://127.0.0.1:9090/actuator/service-registry",
      "templated": false
    },
    "bindings-name": {
      "href": "http://127.0.0.1:9090/actuator/bindings/{name}",
      "templated": true
    },
    "bindings": {
      "href": "http://127.0.0.1:9090/actuator/bindings",
      "templated": false
    },
    "channels": {
      "href": "http://127.0.0.1:9090/actuator/channels",
      "templated": false
    },
    "consul": {
      "href": "http://127.0.0.1:9090/actuator/consul",
      "templated": false
    }
  }
}
```
</details>

両アプリ共に各エンドポイントが有効になっています。一つアクセスして確認してみましょう。


```shell
$ curl http://127.0.0.1:9090/actuator/consul | jq
```

<details><summary>出力例</summary>

```json
{
  "catalogServices": {
    "consul": [
      {
        "id": "29a4d295-f00a-4c55-233e-844482e02912",
        "node": "1c9fcf3d6879",
        "address": "10.5.0.2",
        "datacenter": "dc1",
        "taggedAddresses": {
          "lan": "10.5.0.2",
          "wan": "10.5.0.2"
        },
        "nodeMeta": {
          "consul-network-segment": ""
        },
        "serviceId": "consul",
        "serviceName": "consul",
        "serviceTags": [],
        "serviceAddress": "",
        "serviceMeta": {
          "raft_version": "3",
          "serf_protocol_current": "2",
          "serf_protocol_max": "5",
          "serf_protocol_min": "1",
          "version": "1.6.0"
        },
        "servicePort": 8300,
        "serviceEnableTagOverride": false,
        "createIndex": 267,
        "modifyIndex": 267
      }
    ],
    "web-backend": [
      {
        "id": "29a4d295-f00a-4c55-233e-844482e02912",
        "node": "1c9fcf3d6879",
        "address": "10.5.0.2",
        "datacenter": "dc1",
        "taggedAddresses": {
          "lan": "10.5.0.2",
          "wan": "10.5.0.2"
        },
        "nodeMeta": {
          "consul-network-segment": ""
        },
        "serviceId": "web-backend",
        "serviceName": "web-backend",
        "serviceTags": [
          "secure=false"
        ],
        "serviceAddress": "20884144b872",
        "serviceMeta": {},
        "servicePort": 9090,
        "serviceEnableTagOverride": false,
        "createIndex": 268,
        "modifyIndex": 270
      }
    ],
    "web-front": [
      {
        "id": "29a4d295-f00a-4c55-233e-844482e02912",
        "node": "1c9fcf3d6879",
        "address": "10.5.0.2",
        "datacenter": "dc1",
        "taggedAddresses": {
          "lan": "10.5.0.2",
          "wan": "10.5.0.2"
        },
        "nodeMeta": {
          "consul-network-segment": ""
        },
        "serviceId": "web-front",
        "serviceName": "web-front",
        "serviceTags": [
          "secure=false"
        ],
        "serviceAddress": "b9a130800aa4",
        "serviceMeta": {},
        "servicePort": 8080,
        "serviceEnableTagOverride": false,
        "createIndex": 269,
        "modifyIndex": 272
      }
    ]
  },
  "agentServices": {
    "web-backend": {
      "id": "web-backend",
      "service": "web-backend",
      "tags": [
        "secure=false"
      ],
      "address": "20884144b872",
      "meta": {},
      "port": 9090,
      "enableTagOverride": false,
      "createIndex": null,
      "modifyIndex": null
    },
    "web-front": {
      "id": "web-front",
      "service": "web-front",
      "tags": [
        "secure=false"
      ],
      "address": "b9a130800aa4",
      "meta": {},
      "port": 8080,
      "enableTagOverride": false,
      "createIndex": null,
      "modifyIndex": null
    }
  },
  "catalogNodes": [
    {
      "id": "29a4d295-f00a-4c55-233e-844482e02912",
      "node": "1c9fcf3d6879",
      "address": "10.5.0.2",
      "datacenter": "dc1",
      "taggedAddresses": {
        "lan": "10.5.0.2",
        "wan": "10.5.0.2"
      },
      "meta": {
        "consul-network-segment": ""
      },
      "createIndex": 9,
      "modifyIndex": 268
    }
  ]
}
```
</details>

最後にDocker Composeの標準出力を確認して、デバッグログが出ていることも確認しましょう。以下のようなログが出ることがわかるでしょう。

```
api_1     | 2019-09-15 05:37:43.633 DEBUG 1 --- [nio-9090-exec-4] o.s.web.servlet.DispatcherServlet        : GET "/actuator/health", parameters={}
consul_1  |     2019/09/15 05:37:43 [DEBUG] http: Request GET /v1/catalog/services (314.7µs) from=10.5.0.4:39164
api_1     | 2019-09-15 05:37:43.651 DEBUG 1 --- [nio-9090-exec-4] o.s.w.s.m.m.a.HttpEntityMethodProcessor  : Using 'application/vnd.spring-boot.actuator.v2+json', given [text/plain, text/*, */*] and supported [application/vnd.spring-boot.actuator.v2+json, application/json]
api_1     | 2019-09-15 05:37:43.659 DEBUG 1 --- [nio-9090-exec-4] o.s.w.s.m.m.a.HttpEntityMethodProcessor  : Writing [UP {}]
api_1     | 2019-09-15 05:37:43.662 DEBUG 1 --- [nio-9090-exec-4] o.s.web.servlet.DispatcherServlet        : Completed 200 OK
```

Consulを利用することで設定をアプリからは透過的に変更することができることがわかりました。マイクロサービスなどを採用してアプリケーションの数が増えてきたときこの機能により数十、数百のアプリに対して自動で設定を反映させることが可能です。
