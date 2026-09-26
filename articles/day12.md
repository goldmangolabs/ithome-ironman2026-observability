# Day 12：這個 request 是從 Stable 還是 Canary 來的？

## 今天的工作

1. 提出一個新問題：demo-app 呼叫下游微服務時，下游怎麼知道這是不是金絲雀流量
2. 認識 `Baggage`：另一個由 OTel Agent 自動搬運的 W3C 標準 header
3. 接下來跨服務的路由標籤，一律靠 `Baggage` 自動傳遞，不再手寫 header

Day08-11 我們把「入口分流」這件事做完了。APISIX 收到 request，靠 OPA 判斷、`traffic-split` 執行，準確導向 Stable 或 Canary。今天我們進一步討論，如果 demo-app 收到請求後，還要再呼叫別的服務，我們如何追踪這個 HTTP request 走過的路徑？

## 問題：下游服務不知道自己收到的是不是金絲雀流量

假設 demo-app 要跟下游的微服務 B、C 溝通，會遇到幾個實際問題：

- **資料隔離**：B 要不要把這次的寫入動作導向「影子資料庫」，而不是正式資料庫？沒有標籤，金絲雀測試的假資料會直接污染正式資料。
- **觀測／除錯一致性**：出問題時，能不能在 B、C 的日誌裡搜尋同一個標籤，一次看清楚這次金絲雀實驗在整條呼叫鏈每一站分別發生了什麼事？
- **資源／流量隔離**：B 要不要把金絲雀流量導向獨立的一小群資源，避免還在測試、可能有 Bug 的請求跟正式流量搶資源？

這三件事都建立在同一個前提上：B、C 得先知道「這個 request 是 Stable 還是 Canary」。但 demo-app 呼叫 B 的時候，這個身份要怎麼一起帶過去？

## Baggage：另一個由 OTel Agent 自動搬運的 Header

Day07 我們驗證過 `traceparent`——這個 header 由 OTel Agent 自動攔截、解析。但 `traceparent` 的格式讓它很難夾帶明確 **業務指標**，所以我們很難從它身上看出這個 request 是來自 Stable 還是 Canary。

W3C 另外定義了一個不同用途的標準 header：**Baggage**。格式是逗號分隔的 key-value 清單，官方規格的例子長這樣：

```
baggage: userId=alice,serverNode=DF%2028,isProduction=false
```

跟 `traceparent` 一樣，Baggage 也是由 OTel Agent 自動攔截、自動搬運到下一個出站請求——差別在於 `Baggage` 帶的是 **任意的內容**，使用者可以自己決定要讓什麼內容在 HTTP header 不斷傳下去。這個專案要用它來傳遞一組 key-value：

```
baggage: routing-context=canary
```

### OTel Agent 早就能撈取 Baggage 內容了

Day07 我們讓 demo-app 的 Deployment 設了 `OTEL_PROPAGATORS: "tracecontext,baggage"`：

```yaml
- name: OTEL_PROPAGATORS
  value: "tracecontext,baggage"
```

當時只用到 `tracecontext` 的部分，讓 Agent 解析／傳遞 `traceparent`；`baggage` 這部分雖然也設定好了，但還沒有真正派上用場。今天要做的是讓這個設定真正發揮作用——`baggage` 這個值告訴 Agent 也要用 W3C Baggage 的方式解析／傳遞 `baggage` header，這樣 Agent 就會同時處理兩種 header：`tracecontext` 負責 `traceparent`，`baggage` 負責這個 Baggage header 本身。

只要 demo-app 跟它呼叫的下游服務都裝了 OTel Agent、都設了這個環境變數，這個標籤就會自動一路傳下去，不需要任何一方手寫代碼去讀取、轉發它。

## 這樣就能解決前面那三個問題

回到「問題」那一段列的三件事——B 收到請求時，OTel Agent 會自動讀到 Baggage 裡的 `routing-context` 值，不用 B 自己手動解析 header。實際印出來的日誌會長這樣：

```
2026-08-12 10:00:00 [http-nio-8080-exec-1] INFO  c.example.serviceb.Controller [routing-context=canary] - Handling request
```

B 拿到 `routing-context=canary` 這個值之後，接著：

- **資料隔離**：`routing-context == canary` 就寫進影子資料庫，不動正式資料。
- **觀測／除錯一致性**：把同一個 `routing-context` 值印進 B 自己的日誌，之後查問題時，demo-app、B、C 的日誌可以用同一個關鍵字串起來看。
- **資源／流量隔離**：`routing-context == canary` 就導向另一組資源池，不跟正式流量搶。

三件事的判斷依據都是同一個值，B、C 不需要各自維護一套邏輯去猜這個 request 的身份——這正是「跨服務只靠一個標籤、不用手寫傳遞代碼」這個設計的價值所在。

## Baggage 被 Pod 讀到之後，會發生什麼事

Baggage 不是唯讀的。收到之後，這個 Pod 可以改寫它，甚至刪除某個 key：

- **改寫同名 key**：如果這個 Pod 主動呼叫 Baggage API，把 `routing-context` 設成另一個值，新值會直接取代舊值。不管新值是這個 Pod 自己產生的，還是從上游收到後又被改寫的，一律以最新設定的值為準，繼續往下一跳傳遞。
- **刪除某個 key**：如果這個 Pod 主動呼叫 Baggage API 移除 `routing-context`，這個 key 從這一跳開始就不會再出現在往下一跳送出去的 Baggage 裡。
- **什麼都不做**：這正是這個專案目前 demo-app、middle-app 的實際狀況。Otel Agent 自動附加到出站請求上的 Baggage，跟收到的完全一樣，沒有被動過。

  這個機制馬上就會在 Day13 派上用場：demo-app 呼叫 middle-app-b、middle-app-c 時，`routing-context` 是不是真的原封不動跨三個服務存活，靠的正是今天建立的「Agent 自動搬運、不用手寫傳遞代碼」機制。

  到了 Stage 3（Day15-22，可觀測性階段），Day19 會做類似但不完全一樣的事：用另一個同樣由 OTel Agent 自動搬運的欄位——`trace_id`（不是這裡的 `routing-context`）——把 APISIX 閘道日誌跟 demo-app 應用日誌串起來查（範圍只到 APISIX／demo-app，不含 middle-app）。跟今天建立的 Baggage 機制是同一種設計哲學的延伸，但不是同一個欄位、也不是同一個驗證範圍。

## 架構決策：禁止手寫 header 傳遞邏輯

這個專案從 Day12 開始，增加一條規則：**所有跨服務的路由標籤，一律依賴 OTel Baggage 標準化傳遞**。

## 提醒

- Baggage 帶的 key-value 完全是自訂的，OTel 本身不規定內容——`routing-context=canary` 這個 key 名稱是這個專案自己決定的，不是 Baggage 規格的一部分。
- 今天只解決了「理論上該怎麼做」，`demo-app` 目前還沒有真的呼叫過任何下游服務——這個標籤現在還沒有真正的第二跳可以驗證。

## 參考資料

- [W3C Baggage Specification](https://www.w3.org/TR/baggage/)：Baggage header 的官方格式定義與範例。
- [OTel Baggage API Specification](https://opentelemetry.io/docs/specs/otel/baggage/api/)：同名 key 改寫時「新值必須優先」的官方定義；「刪除後不再往下傳遞」是從 Remove 操作的定義（回傳不含該 key 的新 Baggage）合理推導出的結果，官方文件沒有另外明講這句話。
