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
### OpenTelemetry 和 Dynatrace 一起使用的時機情境

- 涉及完全託管雲服務的複雜雲原生監控環境僅將 OTLP 跟踪暴露為跨度，但不允許部署代理。
- 工程團隊使用的技術，包括第三方函式庫和框架或編程語言，而 Dynatrace 開箱即用未涵蓋也不知如何檢測，但OpenTelemetry 會發出跟踪數據。
- 由使用 OpenTelemetry 自定義檢測的開發團隊以通過項目特定的詳細信息豐富監控數據(例如，添加業務數據或捕獲特定於開發人員的診斷點)

### Demo

- (系統參數說明](https://github.com/open-telemetry/opentelemetry-java/blob/main/sdk-extensions/autoconfigure/README.md)



======================================================






### Dynatrace with OpenTelemetry
![](https://i.imgur.com/1b2mkpd.jpg)

### Send data to Dynatrace with OpenTelemetry

獲取 trace data

- in combination with OneAgent


- Instrument without OneAgent



### Dynatrace free trial
選擇數據中心
![](https://i.imgur.com/43cfk4D.png)
進入Dynatrace平台
![](https://i.imgur.com/hdkV9qi.jpg)
Download Dynatrace OneAgent
![](https://i.imgur.com/H3JAVGa.png)
選擇系統運行的作業系統環境


## 