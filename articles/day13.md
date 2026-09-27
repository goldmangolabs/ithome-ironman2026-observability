# Day 13：驗證 Baggage —— middle-app 登場

## 今天的工作

1. 讓 OPA 的 Rego 除了算出 `x-route-to`，也一起算出 `baggage`——延伸 Day08 已經介紹過的判斷式
2. 引入 middle-app：一支專門用來證明「Baggage 真的能跨服務存活」的下游服務
3. 說明我們打算怎麼驗證、預期會看到什麼結果

Day12 結尾留了一個缺口：「demo-app 目前還沒有真的呼叫過任何下游服務，這個標籤現在還沒有真正的第二跳可以驗證」。今天要把這個缺口補上。我們會設定 OPA service 的 Rego rule，把 baggage 的相關訊息返回給 APISIX 的 `opa` plugin，讓它替 request 加上 `baggage` 的 header。這個值之後會由 demo-app 自己發出的 HTTP request，一路帶到 `middle-app`。

## OPA 怎麼算出 baggage

回顧 Day08 介紹過的第一條 Rego：

```rego
package canary

default allow = true

headers = {
    "x-route-to": "canary"
} {
    input.request.headers["x-canary"] == "true"
}
```

**`headers` 這個變數今天要多算一個東西**：不管這筆 request 有沒有被判定成 Canary，都要算出一個 `baggage` 值，交給下游透傳（Propagation）：

```rego
baggage_val = "routing-context=canary" { is_canary }
baggage_val = "routing-context=stable" { not is_canary }

headers = {
    "x-route-to": "canary",
    "baggage": baggage_val
} {
    is_canary
}

headers = {
    "baggage": baggage_val
} {
    not is_canary
}
```

`baggage_val` 用的是同一種「條件規則」寫法（Day08 介紹過）：`is_canary` 成立就是 `"routing-context=canary"`，不成立就是 `"routing-context=stable"`——兩條規則互斥、剛好覆蓋所有情況，一定會算出一個值，不會有 undefined 的情形。

在 Day08 我們說過：`opa` plugin 的 `send_headers_upstream` 是白名單，OPA 算出來的欄位不主動列進去就會被靜默丟棄。這次要記得把 `baggage` 也加進去：

```yaml
send_headers_upstream:
  - x-route-to
  - baggage
```

到這裡，demo-app 收到的請求 header 裡已經有 `baggage: routing-context=canary`（或 `stable`），OTel Java Agent 會自動把它讀出來、寫進日誌——這是 Day12 已經講過的部分。

兩種情況下，demo-app 實際收到的 request header 長這樣：

```
# 一般流量（沒帶 x-canary），is_canary 不成立
Host: demo-app.example.com
baggage: routing-context=stable
```

```
# QA 流量（帶 x-canary: true），is_canary 成立
Host: demo-app.example.com
x-canary: true
x-route-to: canary
baggage: routing-context=canary
```

`x-canary: true` 是客戶端（QA 人員）自己帶的原始 header，APISIX 不會清掉它（Day08 講過 `send_headers_upstream` 只設定指定的幾個 key，不動其他 header）。`x-route-to` 只有 canary 那組才有——因為 Rego 在 `not is_canary` 的那條規則裡，`headers` 這個物件本來就沒有算出 `x-route-to` 這個 key（Day08 講過的「undefined」）。`baggage` 兩邊都一定有，差別只在值。

## middle-app：讓 Baggage 有真的下游可以到達

單靠 demo-app 自己收到 baggage、印進日誌，只能證明「OPA 算出來的值，demo-app 真的收到了」，證明不了「跨服務」這件事。

middle-app 就是拿來驗證「跨服務」這件事。它有自己獨立的 git repo（`enterprise-gitops-middle-app`）跟獨立的 CI pipeline，跟放 demo-app 原始碼的 repo 是分開的兩個——Day02 建立的 WIF 信任清單也要跟著擴充，才能讓這個新 repo 的 CI 一樣拿到推 image 的權限。它的設計邏輯是：一支極簡的 Spring Boot 服務，可以「接收」與「發出」HTTP request。我們會用同一個 image，搭配不同的 Deployment 部署成不同的服務，靠一個環境變數 `NEXT_HOP_URL` 決定「下一跳打去哪裡」：

- `middle-app-b`：`NEXT_HOP_URL` 指向 `middle-app-c`
- `middle-app-c`：沒有設定 `NEXT_HOP_URL`，是這條鏈路的終點

要多加一個節點（例如 `middle-app-d`），只要多開一份 Deployment manifest，不需要改一行代碼。

demo-app 這邊新增了 `/api/relay` 端點，會真的發出一個對 middle-app 的出站請求：

```java
@GetMapping("/relay")
public ResponseEntity<Map<String, Object>> relay() {
    log.info("Relaying request to middle-app chain");

    if (middleAppUrl == null || middleAppUrl.isBlank()) {
        // 沒設定 MIDDLE_APP_URL，回自己的資訊，不往下打
        ...
    }

    Map<?, ?> downstream = restTemplate.getForObject(middleAppUrl, Map.class);
    // 把下游回傳的結果包進自己的回應
    ...
}
```

middle-app-b 收到請求後（`log.info("Received relayed request")`），走一樣的邏輯，再打給 middle-app-c——三個服務、三段呼叫，串成 `demo-app → middle-app-b → middle-app-c` 一條真的鏈路。

出站呼叫用的是 `RestTemplate`。查過 OTel Java Instrumentation 的官方支援清單，`RestTemplate` 有專屬、明確列名的支援條目（`opentelemetry-spring-web-3.1`）。

**全程三個服務都沒有寫一行手動讀取或轉發 header 的代碼**——baggage 裡的 `routing-context` 完全靠 OTel Agent 自動帶出。demo-app 跟 middle-app 都已經設定好，能夠把收到的 baggage 內容讀出來、印進日誌。

如果一切正常，兩種情境下、三個服務各自預期會印出的日誌大致長這樣（示意，不是真實截圖）：

```
# 一般流量（無 x-canary）→ 全部落在 stable
demo-app-stable:  ... [routing-context=stable] - Relaying request to middle-app chain
middle-app-b:      ... [routing-context=stable] - Received relayed request
middle-app-c:      ... [routing-context=stable] - Received relayed request

# QA 流量（x-canary: true）→ 全部落在 canary
demo-app-canary:  ... [routing-context=canary] - Relaying request to middle-app chain
middle-app-b:      ... [routing-context=canary] - Received relayed request
middle-app-c:      ... [routing-context=canary] - Received relayed request
```

## 我們打算怎麼驗證、預期看到什麼結果

驗證的邏輯很直接：送兩種不同特徵的請求，檢查 demo-app、middle-app-b、middle-app-c 這三個 Pod 各自的 logs，看看 `routing-context` 的值對不對。預期會看到的日誌如下：

- **情境一**：不帶任何 header → 三個 Pod 都應該看到 `routing-context=stable`
- **情境二**：帶 `x-canary: true` → 三個 Pod 都應該看到 `routing-context=canary`

三個 Pod 印出來的 `routing-context` 值如果同一組情境下完全一致，就是「Baggage 真的能跨兩個真實服務跳點存活」這個主張的直接證據。

## 提醒

- middle-app 不是這個專案的主角（主角仍然是 APISIX + OPA + Observability），它存在的目的是給 `Baggage` 一個真的可以驗證的下游，不是要另外教一套微服務開發。

## 參考資料

- [W3C Baggage Specification](https://www.w3.org/TR/baggage/)：文件標題本身就是「Propagation format for distributed context: Baggage」，全篇用「propagate／propagation」描述 Baggage header 該怎麼在服務之間傳遞——這是「透傳」這個詞對應的官方原文用語。
- [OpenTelemetry — Context Propagation](https://opentelemetry.io/docs/concepts/context-propagation/)：官方定義「Propagation is the mechanism that moves context between services and processes. It serializes or deserializes the context object and provides the relevant information to be propagated from one service to another.」
- [OpenTelemetry Java Instrumentation — supported-libraries.md](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/docs/supported-libraries.md)：`RestTemplate` 有專屬支援條目（`opentelemetry-spring-web-3.1`），`RestClient` 沒有——這是這篇選擇 `RestTemplate` 的直接依據。
