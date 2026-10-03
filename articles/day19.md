# Day 19：日誌關聯 — 讓一個 trace_id 同時查到 gateway 與應用的日誌

## 今天的工作

1. 部署 Loki，日誌的儲存與查詢後端
2. 啟用 APISIX 的 `file-logger` 插件，讓 gateway 日誌多帶 `trace_id` 欄位
3. 部署 Promtail，收集 APISIX 跟 demo-app 兩邊 Pod 的 log，推進 Loki
4. 驗證：同一個 `trace_id`，同時在 Loki 查到 APISIX gateway 日誌跟 demo-app 應用日誌

今天我們要收集監控三本柱的最後一個：logs。

## 部署 Loki

Loki 是一個儲存 logs 的工具。

設定檔放在 config repo 的 `manifests/infrastructure/loki/values.yaml`：

```yaml
deploymentMode: SingleBinary

loki:
  auth_enabled: false
  commonConfig:
    replication_factor: 1
  storage:
    type: filesystem
  useTestSchema: true

singleBinary:
  replicas: 1

read:
  replicas: 0
write:
  replicas: 0
backend:
  replicas: 0
```

Loki 支援三種 `deploymentMode`，對應不同的流量規模：

- `SingleBinary` 把收資料、存資料、查資料全部塞進同一個 process 裡跑，資料直接寫本機磁碟。考慮到本專案的規格，我們用這個 mode。
- `SimpleScalable` 把角色拆成 `read`（查詢）、`write`（寫入）、`backend`（壓縮／索引管理）三組各自獨立的 Pod，可以依負載分別擴展
- `Distributed` 拆得更細，每個內部元件都是獨立服務，給極大流量的正式環境用。

## 啟用 APISIX 的 file-logger 插件

APISIX 預設的 access log，是固定格式的一行純文字，裡面 **沒有** `trace_id`。今天要做到「用 `trace_id` 查到 APISIX 這邊的 log」，所以要讓 APISIX 印出一份自訂格式、帶 `trace_id` 的 log。

`file-logger` 是 APISIX 內建插件裡，能做到「自訂輸出格式」這件事的工具，最終匯出的 logs 是 JSON 格式。其中 `log_format` 欄位，可以指定要輸出哪些 nginx 變數，也決定這個 JSON 格式的 logs 要有哪些內容。

跟 Day17 的 `prometheus` 插件、Day18 的 `opentelemetry` 插件一樣，用 `ApisixGlobalRule` 讓所有經過 APISIX 的流量都套用同一組設定：

```yaml
# apisix-file-logger-global-rule.yaml
apiVersion: apisix.apache.org/v2
kind: ApisixGlobalRule
metadata:
  name: global-file-logger
  namespace: <namespace>
spec:
  ingressClassName: apisix
  plugins:
    - name: file-logger
      enable: true
      config:
        path: /dev/stdout
        log_format:
          timestamp: "$time_iso8601"
          client_ip: "$remote_addr"
          host: "$host"
          trace_id: "$opentelemetry_trace_id"
          status: "$status"
          latency: "$request_time"
          upstream: "$upstream_addr"
```

`log_format` 底下每個欄位都是 nginx 內建變數，`trace_id` 讀的是 `$opentelemetry_trace_id` 這個內建變數的值。

但 `$opentelemetry_trace_id` 這個 nginx 變數，**預設不會被賦值**，得先讓這個變數獲得值才行。Day18 啟用的 `opentelemetry` 插件的 `plugin_metadata` 有一個 `set_ngx_var` 布林欄位，預設 `false`，得把這個開關打開，`$opentelemetry_trace_id` 才會真的被賦值：

```yaml
# manifests/infrastructure/apisix/values.yaml（Helm values 摘錄）
ingress-controller:
  gatewayProxy:
    pluginMetadata:
      opentelemetry:
        trace_id_source: "x-request-id"
        resource:
          service.name: "APISIX"
        collector:
          address: "otel-collector.<namespace>.svc.cluster.local:4318"
          request_timeout: 3
        set_ngx_var: true
```

## 部署 Promtail

Promtail 是負責收集和格式化 logs 的工具，是 DaemonSet，透過 Kubernetes Service Discovery 找出每個 Node 上的 Pod，讀取 `/var/log/pods/*/*.log`。K8s 對所有容器的 stdout 都會自動寫成 log 檔，存到 `/var/log/pods/*/*.log`，這包含 APISIX 和 demo-app 的 stdout。我們要讓 Promtail 把收集到的 logs 稍作格式化，再送到 Loki：

設定放在 `manifests/infrastructure/promtail/values.yaml`：

```yaml
config:
  clients:
    - url: http://loki.observability.svc.cluster.local:3100/loki/api/v1/push
  snippets:
    pipelineStages:
      - cri: {}
      - match:
          selector: '{app="apisix"}'
          stages:
            - json:
                expressions:
                  trace_id: trace_id
            - labels:
                trace_id:
      - match:
          selector: '{app="demo-app"}'
          stages:
            - regex:
                expression: '.*trace_id=(?P<trace_id>[0-9a-f]{32}).*'
            - labels:
                trace_id:
```

`pipelineStages` 是一個處理 logs 的 pipeline，這個 pipeline 目前有三站：

1. **`cri: {}`**：第一站，所有 log 不分服務都會先經過這裡——把 container runtime 包在外層的格式拆開，還原成原始的 log 內容。
2. **`match`**：`match` 區塊只對符合 `selector` 的 log 加工處理：

    - `selector` 用的是 `app` 這個標籤，是由 Promtail chart 內建的預設 relabel 規則，把常見的 K8s label（例如 APISIX Pod 實際上是 `app.kubernetes.io/name: apisix`）轉換成這個簡化的 `app` 標籤。
    - `{app="apisix"}` 這一支：上一步啟用的 `file-logger` 插件輸出的是 JSON，`json` stage 直接把整行當 JSON 解析，`expressions` 指定要挖出 `trace_id` 這個欄位，存成暫存變數。
    - `{app="demo-app"}` 這一支：demo-app 的 Logback 輸出的是純文字行，`trace_id=...` 內嵌在訊息裡（Day07 起就有，OTel Java Agent 自動寫進 MDC），沒有 JSON 結構可以解析，改用 `regex` stage，靠 `(?P<trace_id>...)` 這個具名捕獲群組把值從文字裡抓出來。

3. **`labels`**：產生出 Loki 可以使用的 trace_id 格式。
    - `json`／`regex` stage 從 log 裡把 `trace_id` 的值抽取出來——就只是那組字串本身，例如 `b07689219faf1c94f703f03a2d48a287`，存進一個叫 `trace_id` 的暫存變數裡，這時候還沒有 `{trace_id="..."}` 這種查詢語法。
    - `labels` stage 寫 `trace_id:`（冒號後面沒接值），意思是「把這個叫 `trace_id` 的暫存變數，提升成一個同名的 Loki label」——這個 label 的**值**，就是暫存變數當時存的那組字串。做完這一步，`{trace_id="b07689219faf1c94f703f03a2d48a287"}` 才會是一個真的查得到東西的 Loki 查詢。

## 驗證：一個 trace_id 同時查到 gateway 與應用兩邊日誌

驗證策略：送一筆測試請求，從其中一段日誌拿到 `trace_id`，再用同一個 `trace_id` 分別去 Loki 查 `{app="apisix"}` 跟 `{app="demo-app"}` 兩條 log stream——兩邊都查得到才算數，代表 `trace_id` 這個關聯鍵真的把 gateway 跟應用兩層日誌串起來了，不是兩邊各自印各自的、互不相關。

在真的動手查之前，有兩件事要先成立，不然後面查什麼都是空的：`loki`／`promtail` 兩個 ArgoCD Application 要先 Synced/Healthy，`global-file-logger` 這條 `ApisixGlobalRule` 要真的同步進 Admin API——單看 K8s 物件顯示 Synced/Healthy 還不夠，這是 Day17 開始一路踩過的坑，Admin API 端才是真正生效與否的準繩。

前提成立之後，查詢分三步。

### 1. 送出一筆測試請求

```bash
curl -s -o /dev/null -H "Host: demo-app.example.com" -H "x-canary: true" "http://<APISIX 對外 IP>/api/hello"
```

送出後等幾秒，讓 Promtail 把這筆請求留下的 log 推進 Loki。

### 2. 查 APISIX gateway 日誌，取出 trace_id

Loki 沒有掛進 Grafana（見下方「提醒」），查詢得先 port-forward 到它的 HTTP API：

```bash
kubectl port-forward -n observability svc/loki 13100:3100
```

再用 `query_range` API 帶 LogQL 查詢，鎖定剛剛送出請求前後這段時間：

```bash
curl -s -G "http://localhost:13100/loki/api/v1/query_range" \
  --data-urlencode 'query={app="apisix"}' \
  --data-urlencode "start=$(($(date +%s%N) - 60000000000))" \
  --data-urlencode "end=$(date +%s%N)" \
  | jq -r '.data.result[0].values[-1][1]' | jq -r '.trace_id'
```

`start`／`end` 用 nanosecond 時間戳鎖定最近 60 秒，`.data.result[0].values[-1][1]` 取這段時間裡最新一筆 log 的內容（本身是一個 JSON 字串），再用第二個 `jq` 從裡面挖出 `trace_id` 欄位——**非空字串才算過**，空字串代表 `set_ngx_var` 沒開對地方。

### 3. 用同一個 trace_id 查 demo-app 應用日誌

```bash
curl -s -G "http://localhost:13100/loki/api/v1/query_range" \
  --data-urlencode 'query={app="demo-app", trace_id="<上一步取出的值>"}' \
  --data-urlencode "start=$(($(date +%s%N) - 60000000000))" \
  --data-urlencode "end=$(date +%s%N)" \
  | jq -r '.data.result[0].values[0][1]'
```

查得到對應的日誌行，才代表兩邊真的串起來了。

以上三個步驟，簡化自 `scripts/verify-day19-logs.sh` 的真實邏輯——那支腳本額外多做了：先確認 `loki`／`promtail` 兩個 Application 是否 Synced/Healthy、`global-file-logger` 是否真的同步進 Admin API，不用手動一步步下指令。

## 提醒

- Metrics(Day17)／Traces(Day18)／Logs(Day19) 三支柱現在都各自能查了，但還沒有共用一個介面——Day20 會把 Prometheus／Tempo／Loki 一次全部接進 Grafana，今天的驗證仍然是分別對 Loki／Tempo 的 API 下查詢，還不是在同一個 Grafana 介面裡點擊查看。

## 參考資料

- [OpenTelemetry Java Instrumentation — Logger MDC Instrumentation](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/docs/logger-mdc-instrumentation.md)：OTel Java Agent 自動把 `trace_id`／`span_id` 注入 Logback MDC 的官方說明。
- [Apache APISIX — file-logger Plugin](https://apisix.apache.org/docs/apisix/plugins/file-logger/)：`log_format` 欄位與內建 nginx 變數的官方定義。
- [Apache APISIX — opentelemetry Plugin](https://apisix.apache.org/docs/apisix/plugins/opentelemetry/)：`set_ngx_var` 欄位的官方說明，Day18 已經引用過同一份文件。
- [Grafana Loki — Helm Chart values](https://github.com/grafana/loki/blob/main/production/helm/loki/values.yaml)：`deploymentMode`／`read`／`write`／`backend` 等欄位的官方預設值。
- [日誌採集工具選型](../knowledge/日誌採集工具選型.md)：這個專案自己的深度筆記，完整記錄從 ELK 改走 PLG 的決策過程與真實踩坑。
