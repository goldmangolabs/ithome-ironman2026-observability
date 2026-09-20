# Day 06：OTel Java Agent 的零侵入式觀測

## 今天的工作

1. 介紹 OpenTelemetry（OTel）Java Agent 是什麼
2. 幫 Day05 佈署好的 Stable 和 Canary，各自注入 OTel Agent，讓兩者對外都自稱同一個服務、只靠版本標籤區分，方便日後比較兩者的健康度

## 要解決的問題

Day05 我們佈署了 Stable 跟 Canary 兩個版本，每個版本的監控數據要能分開看，也要能放在一起比。

最直覺的做法是在 Spring Boot 代碼裡自己加埋點，但每次要監控新的東西，都得回頭改代碼、重新測試、重新部署，維護成本高，也容易漏埋。

所以今天的做法是：**不改代碼**，用 OTel Java Agent 讓兩個版本各自產生監控數據，並且用同一個服務名稱、只靠版本標籤區分。

## OTel Java Agent：不改代碼就能觀測

### OpenTelemetry 是什麼

- 官方的定義是「observability framework and toolkit」，用來產生、匯出、收集 traces、metrics、logs 這幾種監控資料。
- 它是 CNCF 的專案，跟特定廠商、工具無關（vendor- and tool-agnostic）。
- **它本身不是監控後端**：只負責把資料產生、送出去，資料存在哪裡、怎麼畫圖，是別的工具的事（後面幾天會接 Prometheus、Tempo、Grafana）。

### OTel Java Agent 怎麼做到不改代碼

**先簡單講一點 Java 編譯邏輯：**

- Java 原始碼（`.java`）編譯後會變成**位元組碼（bytecode）**，也就是 `.class` 檔裡的指令。
- JVM 執行程式時，讀的是這些位元組碼，不是我們寫的原始碼。
- 所以「修改位元組碼」就是在 JVM 讀到某個 class 裡的 **method**（class 裡的一個函式，一段做一件事的程式碼）之前，先把這個 method 的指令改一改。改動只發生在記憶體裡，硬碟上的 jar 和原始碼都不動。

**1. `java.lang.instrument` 是什麼**

- 這是 JVM 內建的功能，Java 官方留的一個「後門」。
- 它允許一個叫 **agent** 的小程式，在 class 被載入 JVM 的當下攔住位元組碼，改完再放行。
- Java 官方把這種「在 method 裡插入額外指令」的做法叫 instrumentation（中文常翻成「埋點」或「插樁」），定義是「修改 method 的位元組碼（modification of the byte-codes of methods）」。

**2. `-javaagent` 與 `premain`**

- 啟動指令帶上 `-javaagent:<jar 路徑>`，JVM 就知道要載入這個 agent。
- 啟動順序是：
  1. JVM 初始化
  2. 呼叫 agent 的 `premain`（agent 的進場點）
  3. 才呼叫我們應用程式的 `main`
- 順序很關鍵：`premain` 跑在前面，agent 有機會在 Spring Boot、Tomcat 等 class 被載入之前，先登記「這些 class 進來時通知我」。
- 如果 agent 在應用程式啟動後才來，class 都已經載入完了，就來不及改。

**3. OTel Agent 插入了什麼**

- Agent 認得 Spring MVC、Tomcat、HTTP client 這類常見框架的 method。
- 這些 class 載入時，agent 在特定 method 的**開頭**和**結尾**各插一小段蒐集程式。插進去的邏輯大致是：
  - method 開頭：記下「請求開始了」和開始時間。
  - 原本的 method 照常執行。
  - method 結尾：記下結束時間、狀態碼，算出耗時，然後把這筆資料送出去。
- 比喻：像在每個門口裝一個感應器。門和房間都沒動，只是有人進出時，感應器會自己計時、記錄。

**4. 攔截的位置**

- 進來的請求（inbound requests）：別人打進 demo-app，例如 `/api/hello`。
- 對外的 HTTP 呼叫（outbound HTTP calls）：demo-app 去打其他服務。
- 資料庫呼叫（database calls）：對資料庫下的查詢。
- 這幾個位置剛好涵蓋一個請求「進來、往外走、碰資料庫」的主要路徑，所以能串成一條完整的 Trace。

**5. 為什麼說「零侵入」**

- 我們的 Spring Boot 原始碼一行都沒改，也不用引入任何 OTel 的函式庫。
- 監控邏輯全靠 agent 在 JVM 載入 class 時動態插進去。
- 想關掉，只要把 `JAVA_TOOL_OPTIONS` 拿掉重啟，程式就回到原樣。
- 官方稱這種做法為 zero-code instrumentation，也就是我們說的「零侵入」。

### 要注意的限制

- Agent 只會自動攔截它 **支援清單** 裡的框架跟函式庫，不在清單裡的不會被自動記錄。Day13 選用 `RestTemplate` 而不是 `RestClient`，就是因為前者在官方支援清單裡有明確條目。
- 「零侵入」指的是「不需要改我們的代碼」，不是「什麼都會自動被涵蓋」。

## 怎麼注入

在 Day05 兩份 Deployment 的 container 裡，各自加上環境變數。

**Stable（`demo-app-v1.yaml`）：**

```yaml
env:
  - name: JAVA_TOOL_OPTIONS
    value: "-javaagent:/workspace/opentelemetry-javaagent.jar"
  - name: OTEL_SERVICE_NAME
    value: "demo-app"
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "service.version=v1,deployment.environment=production"
```

**Canary（`demo-app-v2-canary.yaml`）：**

```yaml
env:
  - name: JAVA_TOOL_OPTIONS
    value: "-javaagent:/workspace/opentelemetry-javaagent.jar"
  - name: OTEL_SERVICE_NAME
    value: "demo-app"
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "service.version=v2-canary,deployment.environment=production"
```

兩份幾乎一模一樣，唯一的差異就是 `OTEL_RESOURCE_ATTRIBUTES` 裡的 `service.version`。

## 幾個關鍵點

- `OTEL_SERVICE_NAME` 兩邊都設成同一個 `demo-app`，是為了讓之後 Prometheus／Tempo 能用 `service.version`（`v1` vs `v2-canary`）這個維度，在「同一個服務」底下比較兩個版本的延遲跟錯誤率。如果兩邊 `service.name` 不一樣，追蹤系統會把它們當成兩個完全無關的服務，沒辦法放在同一張圖上比較。
- Agent 的 jar 檔（`opentelemetry-javaagent.jar`）是 `demo-app` 的 Dockerfile 在 build image 時下載的（鎖定版本 v2.29.0），放在 image 的 `/workspace/` 路徑。Deployment 只需要透過 `JAVA_TOOL_OPTIONS` 宣告路徑去載入它，Pod 啟動後 Agent 就跟著啟動。
- `JAVA_TOOL_OPTIONS` 是 JVM 啟動時會讀取的環境變數，放進去的 `-javaagent:...` 效果等同於直接寫在啟動指令上（官方 Getting started 兩種寫法都有列）。用環境變數的好處是不用改 image 裡的啟動指令，由 Deployment 決定要不要啟用 Agent。
- Agent 會把資料送往節點上的 4318 埠，但現在還沒有服務在那裡接收，負責接收的工具「OTel Collector」要到 Day15 才會佈署。

## 參考資料

- [What is OpenTelemetry?](https://opentelemetry.io/docs/what-is-opentelemetry/)：OTel 的官方定義、CNCF 專案身分、「不是監控後端」的說明
- [Zero-code instrumentation：Java agent](https://opentelemetry.io/docs/zero-code/java/agent/)：「dynamically injects bytecode…」與攔截位置的原文
- [Java agent Getting started](https://opentelemetry.io/docs/zero-code/java/agent/getting-started/)：`-javaagent` 與 `JAVA_TOOL_OPTIONS` 兩種啟用寫法
- [java.lang.instrument（Java SE 17）](https://docs.oracle.com/en/java/javase/17/docs/api/java.instrument/java/lang/instrument/package-summary.html)：「The mechanism for instrumentation is modification of the byte-codes of methods」與 `premain` 的呼叫時機
