# Day 07：驗證請求真的到 Stable，OTel Agent 也真的在運作

## 今天的工作

1. 用 curl 打 APISIX，從終端機收到的回覆內容，驗證是 Stable 在回應
2. 用一組自製的 `traceparent` header，驗證 OTel Java Agent 有攔截並解析它

## 驗證一：request 真的到達 Stable

拿到 APISIX load balancer 的外部 IP 後，帶上 `ApisixRoute` 比對用的假網域直接打過去：

```bash
curl -H "Host: demo-app.example.com" http://<APISIX_LoadBalancer_IP>/api/hello
```
  
終端機會收到：

```json
{"version":"v1.0.0","msg":"Hello from demo-app"}
```

`version` 這欄回的是 `v1.0.0`。這個值是 demo-app 啟動時從環境變數 `APP_VERSION` 讀來的，由各自的 Deployment 設定：Stable 設 `v1.0.0`，Canary 設 `v2.0.0-canary`。所以看到 `v1.0.0`，就能確定是 **Stable** Pod 回應的。

## 驗證二：OTel Java Agent 真的有攔截、解析 request

### 我們用 traceparent header 來驗證

`traceparent` 是 W3C Trace Context 規格定義的標準 HTTP header，用來告訴下一個服務：這個請求屬於哪一次追蹤、上一步是誰，讓各服務能把自己那一段接到同一條追蹤鏈上。格式是四段用 `-` 分隔的十六進位字串，官方規格給的範例長這樣：

```
<version>-<trace-id>-<parent-id>-<trace-flags>
00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

- `version`（2 碼）：格式版本，目前固定是 `00`
- `trace-id`（32 碼）：這次追蹤的唯一識別碼，同一條呼叫鏈上的每一段都共用同一個值
- `parent-id`（16 碼）：上一段（parent span）的識別碼
- `trace-flags`（2 碼）：控制旗標，`01` 代表這筆請求要被取樣（sampled）

### 讓 OTel Java Agent 提取 traceparent header

demo-app 的 Deployment 裡有這個環境變數：

```yaml
- name: OTEL_PROPAGATORS
  value: "tracecontext,baggage"
```

`tracecontext` 這個值告訴 Agent 要用 W3C Trace Context 的方式解析／傳遞 `traceparent`；`baggage` 是另一個 W3C 標準，讓 Agent 也一併解析／傳遞 `baggage` 這個 header（Day13 會用到，OPA 算出的 `routing-context` 值就是靠這個機制透傳給下游服務）。這其實就是官方預設值本身，寫在這裡是明確表達意圖，不是覆寫預設。

### 驗證 OTel Java Agent 有沒有好好工作

驗證邏輯：用 `openssl` 產生一組獨一無二的 `trace-id`，透過 `traceparent` header 送給 demo-app，再去查 Pod 的 log 裡有沒有出現這組 ID。

先產生 ID 並送出請求：

```bash
TRACE_ID=$(openssl rand -hex 16)
curl -H "Host: demo-app.example.com" \
     -H "traceparent: 00-${TRACE_ID}-0000000000000001-01" \
     http://<APISIX_EXTERNAL_IP>/api/hello
```

送出之後，去撈 `demo-app-stable` Pod 的 log：

```bash
kubectl logs -l app=demo-app,track=stable | grep "${TRACE_ID}"
```

如果撈得到，就代表 OTel Java Agent 真的有攔截這個請求、解析 `traceparent`、把 `trace_id` 寫進日誌。

### 「撈得到 log」這件事本身就是 OTel Java Agent 運作正常的證據

我們的 `demo-app` 原始碼裡完全沒有寫任何「讀取 HTTP header」的邏輯——`/api/hello` 唯一一行 log 是固定死的字串 `log.info("Processing hello request")`，代碼裡沒有任何一行去讀 `traceparent`，也沒有手動把 trace-id 塞進日誌。這組自製的 `TRACE_ID` 之所以能出現在日誌裡，中間發生了兩件事，而且都是 OTel Java Agent 自動做的：

1. Agent 攔截這筆請求時，解析 `traceparent` header，把裡面的 `trace-id` 當成這個請求所屬的 trace context。
2. Agent 內建的 Logback 整合功能，會把目前這個 trace context 的 `trace_id`／`span_id` 自動寫進 Logback 的 MDC——這是**預設行為，不用額外設定**。`demo-app` 的 `logback-spring.xml` pattern 裡寫了 `%X{trace_id}`，這是 Logback 語法「去 MDC 撈這個 key 印出來」，這就是每一行 log 最後能印出 trace-id 的直接原因。

換句話說，如果 Agent 沒有正常運作，這兩步都不會發生，這組自製的 trace-id **不會以任何形式出現在日誌裡**——不是「換個方式出現」，是代碼裡根本沒有其他管道能讓它出現。

## 小結

這兩個驗證合起來，才真正確認了 Day04、Day05、Day06 三天佈署的東西都如預期運作：路由能把 request 送到正確的版本，而且零侵入式的觀測機制真的在背後默默工作。

## 參考資料

- [W3C Trace Context — traceparent header](https://www.w3.org/TR/trace-context/#traceparent-header)：官方定義 `traceparent` 的格式（`version-trace_id-parent_id-trace_flags`）與範例，這是這篇自製 `traceparent` header 時採用的格式依據。
- [OpenTelemetry Java — Agent Configuration](https://opentelemetry.io/docs/languages/java/configuration/#properties-general)：官方文件列出 `OTEL_PROPAGATORS` 的預設值是 `tracecontext,baggage`——這是「demo-app 的設定其實只是明確寫出預設值」這個說法的依據。
- [OpenTelemetry Java Instrumentation — logback-mdc-1.0 (javaagent)](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/instrumentation/logback/logback-mdc-1.0/javaagent/README.md)：官方文件說明 Agent 預設就會把目前的 `trace_id`／`span_id` 寫進 Logback 的 MDC，不需要額外設定——這是「log 裡撈得到 trace-id」這件事背後的機制依據。
