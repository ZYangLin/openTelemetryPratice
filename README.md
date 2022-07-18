{%hackmd theme-dark %}
[TOC]
# OpenTelemetry 技術分享
在開始主題之前需要先提到與系統監控領域息息相關的詞 "Observability(可觀測性)"。 可觀測性是指系統可以由其外部的輸出來推斷內部狀態訊息。 可觀測性分為以下三個面向(aka 可觀測性的三根支柱):
- **Metrics**: 以數值化的方式表達系統狀態資訊，譬如 CPU 負載率、記憶體使用量
- **Logs**: 系統對於事件發生當下所紀錄的訊息
- **Traces**: 將事件關聯成一個從頭到尾的流程，譬如一個 request 到 response 這之間所有執行到的服務、觸發的方法甚至使用的 SQL 指令、DB 系統
:::info
如上所述，可分成三個面向來做深入探討和技術的使用，而本篇文章都以trace為主
:::


其中 Traces 的主要特性之一就是跨服務邊界來關聯事件，然而在過往單體式架構下大家多專注於 log 的記錄和 metrics 的呈現。 可是單體式的架構下隨著系統的擴張，維護開始變得不易，甚至連系統的部署、運行也日趨耗時，大家因此將系統轉型成微服務架構來解決問題，與此同時分散式環境下 traces 的議題也開始被大家所重視。 

:nerd_face:系統服務的監控最好是三個面向都考慮進去，而非只專注於某一面向，這樣才能達到問題發生前的預警以及發生後的快速排除。






## Introduction OpenTelemetry
### What is OpenTelemetry 

OpenTelemetry 其實不是一個憑空誕生的專案，如前面所說 traces 面向的監控開始被大家所重視，所以在 2019 年大家主要以 Google 的 OpenCensus 以及 CNCF 的 OpenTracing 這兩大專案來達到 distributed tracing 目的。 而這兩個專案各有好用的地方和各自的擁護者，誰也無法取代誰。 最後在2019年合併成為了 OpenTelemerty 並由 CNCF 孵化的專案。


![](https://i.imgur.com/c326Qcl.png "ref. Construction and Operation of Observability Platform using Istio")

並且也是 CNCF 上第二活躍的專案。

![](https://i.imgur.com/ArMQD74.png "ref. CNCF Dev Status")

而 OpenTelemetry 從名稱來看主要分為 Open+Telemetry，其中 Telemetry 又是兩希臘古字 Tele(遠程)+Metron(測量)組成，因此 OpenTelemetry 為開源的**遙測**技術，且產生的資料又稱為遙測數據(Telemetry data)。 遙測技術的應用其實很廣泛，像是現在很多人在用的智能手錶也是將我們的資訊透過手錶收集並呈現數據讓我們監控自己的身體狀況。


關於 OpenTelemetry ，官網中也開宗明義寫著: 
: OpenTelemetry is a collection of tools, APIs, and SDKs. Use it to instrument, generate, collect, and export telemetry data (metrics, logs, and traces) to help you analyze your software’s performance and behavior.

就是說 OpenTelemetry 是由 tools、 APIs、 SDKs 所組成的集合，它會進行 instrument, generate, collect, 以及 export 等動作，最後會導出 metrics, logs, and traces 這些具可觀測性的遙測數據，來幫助我們分析系統效能與行為。

:::info
在以往我們總會需要配合後端分析工具的不同，來改變在client的代碼以產生對應的數據格式。 OpenTelemtry 的出現讓依賴反轉(dependency inversion)了過來，我們統一在使用openTelemtry提供的agent變可以抽換甚至使用多個後端工具、平台
:::




### OpenTelemetry component
![](https://i.imgur.com/cSQiaFT.png)
OpenTelemetry Component 可分成三個部分，
- Client
    - API: 定義用於 tracing, metrics, and logging data 的資料類型
    - SDK: API的實作以及設定後續資料要做的處理
    - Exporters: 遙測數據產生後需要發送到指定的後端，此時有兩種方式，第一種是直接發送過去另一種是可以通過collector做為代理
- Collector: collector 並非必要的存在，也可以在前面 client 就直接輸出資料到指定的後端分析工具(像是 jaeger, zipkin .etc)
    - Receivers:  從各種來源接收各種格式的遙測數據，並在接收後轉換為OTLP格式進行處理(otlp 為預設傳輸格式，另外還有jeager, zipkin 的傳輸格式)
    - Processors: 將數據進行清洗及過濾
    - Exporters: 將數據傳送到指定後端
- Backend Storage: 遙測數據最後傳送到的地方，不同面向有不同的工具使用，當然也有些工具平台可以將三個面向做結合來呈現。
    - **Metrics**: Prometheous
    - **Logs**: 
    - **Traces**: Jaeger, Zipkin
    - [Distributions vendors](https://opentelemetry.io/vendors/)

#### OpenTelemetry Instrumentation
OpenTelemetry 會先需要對程式碼進行 instrumentation 後才能產生後續的遙測數據。而 openTelemetry 的檢測分成 Auto-Instrumentation, Manual Instrumentation

- Auto-Instrumentation
openTelemtry 會針對@RestController進行aop(corss-cutting concerns)發送前進行log、trace等資訊的埋入，此外也可以宣告annotation @WithSprin, @SpanAttribute 做客製化的span來使用。

- Manual Instrumentation 
在程式碼內使用openTelemtry sdk 提供的api方法，宣告相關trace, log 等實例
```java=
Span span = tracer.spanBuilder("my span").startSpan();
// put the span into the current Context
try {
  // do something...
} finally {
    span.end();
}
```

### Context and Propagation
在分散式環境下跨服務邊界將一系列操作行為串連起來追蹤便是distrubted tracing的特性，而 openTelemetry 會藉由共享在context object 中以key-value的格式儲存的 metadata 來達到該目的。

![](https://i.imgur.com/vbPNV2n.png)


- Context
    - Span context: 跨邊界追蹤時所需要的資訊
        - Trace ID:  the trace identifier of the span context.
        - Span ID: the span identifier of the span context.
        - Trace flags: the trace options for the span context.
        - Trace state: the trace state for the span context.
    - Correlation context: 非必要的組件 表示用戶自定義屬性
        - Customer id
        - Host name
- Propagation: 跟context綁定來傳播的機制 通常透過 http headers來傳播


## OpenTelemetry with Dynatrace
### OpenTelemetry 和 Dynatrace 一起使用的時機與限制
#### 時機
- 當 Dynatrace 開箱即用沒有支援的技術時可以搭配 openTelemtry 來拓展監控的範圍
- OpenTelemtry 支援span, metrics的客製化，因此兩者搭配起來可以使監控遙測的數據更加豐富化

#### 限制
- OneAgent replaces both the API-only and the SDK tracer. With OpenTelemetry Java enabled, existing tracers (such as Jaeger) will no longer see spans.
- When both OneAgent and OpenTelemetry sensors are present for the same technology, you may experience the following limitations:
    - Duplicate nodes in distributed traces
    - Additional overhead
- The OpenTelemetry Java sensor doesn't capture array-type attributes.

### openTelemetry 與 Dynatrace 的相互使用有兩種模式
#### In combination with OneAgent
![](https://i.imgur.com/Uc2xvVA.png)

在 OneAgent code module 中啟用 OpenTelemetry integration
- OneAgent 會自動挖掘 OpenTelemetry custom or pre-instrumentation 所產生的遙測數據並發送到 Dynatrace 平台。
- 在 Dynatrace 介面中去啟用 OpenTelemetry Java 選項
![](https://i.imgur.com/pEDWdb6.jpg)






#### Instrument without OneAgent
![](https://i.imgur.com/6Y9psq9.png)

該模式類似於先使用openTelemetry後導入Dyanatrace。用於無法由 
1. OneAgent 代碼模塊檢測的服務的情況。 
2. 或者要監控的組件已經以 OpenTelemetry 格式 (OTLP) expose 跟踪數據時


可以通過 Dynatrace ActiveGate 上提供的 API 將 OTLP 格式的 OpenTelemetry 跟踪數據（跟踪和跨度）發送到 Dynatrace。攝取的跨度集成到 PurePath® 分佈式跟踪中。

如果您要監控的服務已由 OneAgent 檢測，並且您希望通過 OpenTelemetry 獲得更多見解，請查看 OneAgent OpenTracing 和 OpenTelemetry 支持https://www.dynatrace.com/support/help/extend-dynatrace/extend-tracing/opentracing。

How to instrument and send trace data to Dynatrace
1. 使用 OpenTelemetry 檢測您的服務	
    1. 使用openTelemetry agent
    2. 使用環境變數指定protocol
    3. 在 Dynatrace 查看監控數據是否導出
2. 創建身份驗證令牌
3. 配置 OpenTelemetry 導出器
    1. 將 URL 和 Authorization token 寫在 otel-collector-config.yml
    ![](https://i.imgur.com/L4sYLB2.jpg)
    
    api-token 設定於 Dynatrace SaaS 和 Dynatrace Managed 的 URL 會不同需注意
    - Dynatrace SaaS:  https://{your-environment-id}.live.dynatrace.com/api/v2/otlp/v1/traces
    - Dynatrace Managed:  https://{your-domain}/e/{your-environment-id}/api/v2/otlp/v1/traces

4. 登入Dynatrace，驗證跟踪是否傳送到 Dynatrace
5. (可選)配置數據捕獲以滿足隱私要求
![](https://i.imgur.com/9TCWK87.png)

6. (可選)通過指標檢查攝取的跟踪量






## Demo

### Demo1

Step1. Have a project
我使用網路上的open proj "[PetClinic](https://github.com/spring-projects/spring-petclinic)"

Step2. Download [Java agent](https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases)

Step3. Export to [Jaeger](https://www.jaegertracing.io/docs/1.6/getting-started/)
> docker run -d --name jaeger  -e COLLECTOR_ZIPKIN_HOST_PORT=:9411  -p 5775:5775/udp  -p 6831:6831/udp  -p 6832:6832/udp  -p 5778:5778  -p 16686:16686  -p 14250:14250  -p 14268:14268  -p 14269:14269  -p 9411:9411  jaegertracing/all-in-one:1.32


Step4. run command: 
> OTEL_SERVICE_NAME=my-service OTEL_TRACES_EXPORTER=jaeger OTEL_EXPORTER_JAEGER_ENDPOINT=http://localhost:14250 java -javaagent:./opentelemetry-javaagent.jar -jar target/*.jar



Step5. view [local webSite](http://localhost:9000/owners?lastName=) and [Jaeger UI](http://localhost:16686/)


- [系統參數說明](https://github.com/open-telemetry/opentelemetry-java/blob/main/sdk-extensions/autoconfigure/README.md)


### Demo2

Step1. Have a project(接續前一個範例使用的專案)

Step2. Download Java agent

Step3. docker-compose up containers

Step4. run command: 
> OTEL_RESOURCE_ATTRIBUTES=service.name=my-service OTEL_TRACES_EXPORTER=otlp OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://localhost:4317 OTEL_EXPORTER_OTLP_PROTOCOL=grpc OTEL_EXPORTER_OTLP_TRACES_PROTOCOL=grpc  java -javaagent:./opentelemetry-javaagent.jar -jar target/*.jar

Step5. view tools([Jaeger](http://localhost:16686/search), [Prometheus](http://localhost:9090/graph), [Grafana](http://localhost:3000/?orgId=1), ) and [website](http://localhost:9000/)


#### Collector 設定 
**docker-compose.yml**
```dockerfile= docker-compose.yml
version: "3"

services:
  mysql:
    image: mysql:5.7
    platform: "linux/amd64"
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=
      - MYSQL_ALLOW_EMPTY_PASSWORD=true
      - MYSQL_USER=petclinic
      - MYSQL_PASSWORD=petclinic
      - MYSQL_DATABASE=petclinic
    volumes:
      - "./conf.d:/etc/mysql/conf.d:ro"
  postgres:
    image: postgres:14.1
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_PASSWORD=petclinic
      - POSTGRES_USER=petclinic
      - POSTGRES_DB=petclinic
  node-exporter:
    image: prom/node-exporter:v1.2.2
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
      - '--no-collector.arp'
      - '--no-collector.netstat'
      - '--no-collector.netdev'
      - '--no-collector.softnet'

##################################
# OpenTelemetry      
  # Jaeger
  jaeger-all-in-one:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "14250:14250"
  
  # Zipkin
  zipkin-all-in-one:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"
  
  # Prometheus
  prometheus:
    container_name: prometheus
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  # Grafana
  grafana:
    container_name: grafana
    image: grafana/grafana
    ports:
      - "3000:3000"

  # Collector
  otel-collector:
    image: otel/opentelemetry-collector-contrib-dev:latest
    command: ["--config=/etc/otel-collector-config.yml"]
    volumes:
      - ./otel-collector-config.yml:/etc/otel-collector-config.yml
    ports:
      - "1888:1888"   # pprof extension
      - "8888:8888"   # Prometheus metrics exposed by the collector
      - "8889:8889"   # Prometheus exporter metrics
      - "13133:13133" # health_check extension
      - "4317:4317"   # OTLP gRPC receiver
      - "4318:4318"
      - "55679:55679" # zpages extension
    depends_on:
      - jaeger-all-in-one
      - zipkin-all-in-one
      - prometheus
```
---
**otel-collector-config.yml**
```yaml= otel-collector-config.yml
#負責接受數據，格式有 grpc, jaeger, zipkin
receivers:
  otlp:
    protocols:
      grpc:
      #http:
#負責資料清洗，過濾
processors:
  batch:

#負責設定最後遙測數據要傳送到哪邊去
exporters:
  # OTLP
  otlp:
    endpoint: otelcol:4317
  # Data sources: traces
  zipkin:
    endpoint: "http://zipkin-all-in-one:9411/api/v2/spans"
  # Data sources: traces, metrics, logs
  logging:
    loglevel: debug
  # Data sources: traces
  jaeger:
    endpoint: "jaeger-all-in-one:14250"
    tls:
      insecure: true
  # Prometheus
  prometheus:
    endpoint: "0.0.0.0:8889"
    const_labels:
      label1: value1

extensions:
  health_check:
  pprof:
  zpages:

#在上面將要接收、處理、和產出的參數設定後，最後要在這邊做最後的enable
service:
  extensions: [health_check,pprof,zpages]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [logging, jaeger]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [logging, prometheus]
```

#### Jaeger
初始畫面，在此一個直觀的觀察方式是右上角的圓點越大顆代表所花費的時間越多，通常也是要處理的問題所在處
![](https://i.imgur.com/LttoSj5.jpg)

針對每個tree進入，會看到瀑布狀的鏈路追蹤
![](https://i.imgur.com/VqzCESn.jpg)

jaeger 內除了 Trace Timeline 也有 json 可以看
:::info
OpenTelemetry 中，一個trace可代表一個rq到rp間的所有操作。所以一個tree裡面會有多個span，而所有spans共用一個 traceID，來達到將他們標示為一個tree的概念。另外spans會有父子以及兄弟的關係，child-span會標註ref spanID的方式來連接到父親span
:::

```json=
{
    "data": [
        {
            "traceID": "edc74b88be0af659a08043f60130bb77",
            "spans": [
                {
                    "traceID": "edc74b88be0af659a08043f60130bb77",
                    "spanID": "9734abd03fb3038f",
                    "operationName": "OwnerController.initFindForm",
                    "references": [
                        {
                            "refType": "CHILD_OF",
                            "traceID": "edc74b88be0af659a08043f60130bb77",
                            "spanID": "e16536121b7a1d19"
                        }
                    ],
                    "startTime": 1658132533091033,
                    "duration": 9185,
                    "tags": [
                        {
                            "key": "otel.library.name",
                            "type": "string",
                            "value": "io.opentelemetry.spring-webmvc-3.1"
                        },
                        {
                            "key": "otel.library.version",
                            "type": "string",
                            "value": "1.14.0-alpha"
                        },
                        {
                            "key": "thread.id",
                            "type": "int64",
                            "value": 137
                        },
                        {
                            "key": "thread.name",
                            "type": "string",
                            "value": "http-nio-9000-exec-4"
                        },
                        {
                            "key": "span.kind",
                            "type": "string",
                            "value": "internal"
                        },
                        {
                            "key": "internal.span.format",
                            "type": "string",
                            "value": "proto"
                        }
                    ],
                    "logs": [],
                    "processID": "p1",
                    "warnings": null
                },
                {
                    "traceID": "edc74b88be0af659a08043f60130bb77",
                    "spanID": "94160f40b9405837",
                    "operationName": "Render owners/findOwners",
                    "references": [
                        {
                            "refType": "CHILD_OF",
                            "traceID": "edc74b88be0af659a08043f60130bb77",
                            "spanID": "e16536121b7a1d19"
                        }
                    ],
                    "startTime": 1658132533100310,
                    "duration": 152087,
                    "tags": [
                        {
                            "key": "otel.library.name",
                            "type": "string",
                            "value": "io.opentelemetry.spring-webmvc-3.1"
                        },
                        {
                            "key": "otel.library.version",
                            "type": "string",
                            "value": "1.14.0-alpha"
                        },
                        {
                            "key": "thread.id",
                            "type": "int64",
                            "value": 137
                        },
                        {
                            "key": "thread.name",
                            "type": "string",
                            "value": "http-nio-9000-exec-4"
                        },
                        {
                            "key": "span.kind",
                            "type": "string",
                            "value": "internal"
                        },
                        {
                            "key": "internal.span.format",
                            "type": "string",
                            "value": "proto"
                        }
                    ],
                    "logs": [],
                    "processID": "p1",
                    "warnings": null
                },
                {
                    "traceID": "edc74b88be0af659a08043f60130bb77",
                    "spanID": "e16536121b7a1d19",
                    "operationName": "/owners/find",
                    "references": [],
                    "startTime": 1658132533075689,
                    "duration": 178314,
                    "tags": [
                        {
                            "key": "otel.library.name",
                            "type": "string",
                            "value": "io.opentelemetry.tomcat-7.0"
                        },
                        {
                            "key": "otel.library.version",
                            "type": "string",
                            "value": "1.14.0-alpha"
                        },
                        {
                            "key": "http.target",
                            "type": "string",
                            "value": "/owners/find"
                        },
                        {
                            "key": "http.flavor",
                            "type": "string",
                            "value": "1.1"
                        },
                        {
                            "key": "thread.id",
                            "type": "int64",
                            "value": 137
                        },
                        {
                            "key": "net.transport",
                            "type": "string",
                            "value": "ip_tcp"
                        },
                        {
                            "key": "net.peer.ip",
                            "type": "string",
                            "value": "0:0:0:0:0:0:0:1"
                        },
                        {
                            "key": "thread.name",
                            "type": "string",
                            "value": "http-nio-9000-exec-4"
                        },
                        {
                            "key": "http.host",
                            "type": "string",
                            "value": "localhost:9000"
                        },
                        {
                            "key": "http.status_code",
                            "type": "int64",
                            "value": 200
                        },
                        {
                            "key": "http.route",
                            "type": "string",
                            "value": "/owners/find"
                        },
                        {
                            "key": "http.user_agent",
                            "type": "string",
                            "value": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.0.0 Safari/537.36"
                        },
                        {
                            "key": "http.method",
                            "type": "string",
                            "value": "GET"
                        },
                        {
                            "key": "net.peer.port",
                            "type": "int64",
                            "value": 50777
                        },
                        {
                            "key": "http.scheme",
                            "type": "string",
                            "value": "http"
                        },
                        {
                            "key": "span.kind",
                            "type": "string",
                            "value": "server"
                        },
                        {
                            "key": "internal.span.format",
                            "type": "string",
                            "value": "proto"
                        }
                    ],
                    "logs": [],
                    "processID": "p1",
                    "warnings": null
                }
            ],
            "processes": {
                "p1": {
                    "serviceName": "my-service",
                    "tags": [
                        {
                            "key": "host.arch",
                            "type": "string",
                            "value": "x86_64"
                        },
                        {
                            "key": "host.name",
                            "type": "string",
                            "value": "LINs-MacBook-Pro.local"
                        },
                        {
                            "key": "os.description",
                            "type": "string",
                            "value": "Mac OS X 12.3"
                        },
                        {
                            "key": "os.type",
                            "type": "string",
                            "value": "darwin"
                        },
                        {
                            "key": "process.command_line",
                            "type": "string",
                            "value": "/Library/Java/JavaVirtualMachines/jdk-11.0.15+10 2/Contents/Home:bin:java -javaagent:./opentelemetry-javaagent.jar"
                        },
                        {
                            "key": "process.executable.path",
                            "type": "string",
                            "value": "/Library/Java/JavaVirtualMachines/jdk-11.0.15+10 2/Contents/Home:bin:java"
                        },
                        {
                            "key": "process.pid",
                            "type": "int64",
                            "value": 61090
                        },
                        {
                            "key": "process.runtime.description",
                            "type": "string",
                            "value": "Eclipse Adoptium OpenJDK 64-Bit Server VM 11.0.15+10"
                        },
                        {
                            "key": "process.runtime.name",
                            "type": "string",
                            "value": "OpenJDK Runtime Environment"
                        },
                        {
                            "key": "process.runtime.version",
                            "type": "string",
                            "value": "11.0.15+10"
                        },
                        {
                            "key": "telemetry.auto.version",
                            "type": "string",
                            "value": "1.14.0"
                        },
                        {
                            "key": "telemetry.sdk.language",
                            "type": "string",
                            "value": "java"
                        },
                        {
                            "key": "telemetry.sdk.name",
                            "type": "string",
                            "value": "opentelemetry"
                        },
                        {
                            "key": "telemetry.sdk.version",
                            "type": "string",
                            "value": "1.14.0"
                        }
                    ]
                }
            },
            "warnings": null
        }
    ],
    "total": 0,
    "limit": 0,
    "offset": 0,
    "errors": null
}
```


#### Prometheus
Prometheus 為監控數據裡面metrics的那一面向，所以所顯示的資訊偏向系統cpu或記憶體空間等。 Prometheus 可搭配系統監測，讓我們監控系統健康度，並設定觸發警示條件通知我們來處理問題。

**prometheus.yml**
```yaml= prometheus.yml
# 全域設定
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

scrape_configs:
  - job_name: 'otel-collector'
    scrape_interval: 10s
    static_configs:
      - targets: ['otel-collector:8889']
      - targets: ['otel-collector:8888']
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: node
    static_configs:
      - targets: ["node-exporter:9100"]
  
```

#### Grafana
> 感覺偏向一個可以彙整眾多工具的平台，在grafana中可以藉由設定data source的方式來轉到出 jaeger, prometheus等遙測數據到此處

![](https://i.imgur.com/KUZBIbj.png)

Jaeger 這邊的ip需要進入容器ifconfig出自己在容器內的ip，而不是local host ip
![](https://i.imgur.com/MB8gVE0.png)

Prometheus 也是
![](https://i.imgur.com/X0scX0u.png)

最終， jaeger + Prometheus 大概長這樣。 當然還可以加入其他很多data source
![](https://i.imgur.com/w01L5DK.jpg)


## 結論

以上主要針對openTelemetry的使用，並沒有太去深入了解那些後端的分析工具有哪些功能和相關的設定.....

OpenTelemetry 可以去 exporter 遙測數據到負責 logs, traces, metrics 的各個分析工具來呈現出最貼近我們需求的，最全面的監控服務，並且在更動這些後端時不用怕需要去動到埋在client的代碼。

或者，也可以考慮使用那些已經把 log, trace, metric 包在一起使用的工具平台，並且如果系統未來不太會有抽換後端監控服務的問題的話，我覺得好像就也不一定要使用openTelemetry了 :thinking_face: :thinking_face: :thinking_face: 



###### tags: `openTelemetry`, `Dynatrace`, `Jaeger`, `Y.`
