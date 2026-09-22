# Day 08：認識 OPA——只用 JSON 對話的決策引擎

## 今天的工作

1. 認識 OPA：一個只用 JSON 對話的決策引擎，以及它「算不出答案，就是完全沒有這個欄位」的特殊行為
2. 幫 APISIX 接上 `opa` plugin，這是兩者之間的溝通橋樑
3. 讀懂第一條 Rego 判斷式，理解它怎麼「算出」或「算不出」一個值

今天正式進入第二階段。Day01-07 我們讓 APISIX 把所有流量都固定送去 Stable，接下來要讓它能依條件動態切到 Canary——負責做這個判斷的，就是今天要介紹的 OPA。

## OPA 是什麼

### APISIX 是大老闆，OPA 是查手冊的小秘書

**APISIX（大老闆）**：手上有大量路由跟後端（backends）設定，收到 request 時負責「確實執行」——放行、轉發、修改 Http header，但 **不負責思考複雜邏輯**。

**OPA（小秘書）**：一個獨立的微服務，收到 APISIX 丟過來的 request 特徵 (JSON 格式)，翻自己手上的規則（Rego 語法）算出一個結果，回傳（JSON 格式）給 APISIX。**它完全不知道、也不關心後端服務實際在哪裡**——這是最容易被誤解的地方：很多人以為 OPA 既然是「決策引擎」，就一定知道流量的最終去向，但它其實只是個單純的計算機，只負責回答問題，不負責執行。

### OPA 用 JSON 與 APISIX 溝通

OPA 跟 APISIX 之間溝通的內容，使用 JSON 格式。APISIX 把 request 特徵打包成 JSON 丟給 OPA（這叫 `input`），OPA 算完之後回一包 JSON 給 APISIX 的 `opa` plugin。在收到 OPA service 回傳的 JSON 之後，會 **調整 Http header**。

OPA 的規則（有時也稱"策略"或"判斷式"），是用 Rego 語法寫的：

```rego
package canary

default allow = true

headers = {
    "x-route-to": "canary"
} {
    input.request.headers["x-canary"] == "true"
}
```

`headers = {...} { 條件 }` 是一條 **條件規則**：只有大括號裡的條件（`input.request.headers["x-canary"] == "true"`）成立，`headers` 這個變數才會被賦值成前面那個物件。Rego 沒有 `else` 分支——條件不成立，`headers` 就是「undefined」，完全不存在，不是被賦值成 `null` 或空物件 `{}`。  

`default allow = true` 確保這個 OPA 回傳的 JSON 有 `"allow": true`。  
對 `opa` plugin 來說，它把 allow 當成「要不要讓這筆 request 通過」的 **第一順位** 判斷依據。如果 `allow = false` 或者根本沒有 `allow`，`opa` plugin 直接判定拒絕 request，變成 403。

反映到 OPA 實際回傳給 APISIX 的 JSON：

```
帶 x-canary: true 時 →  { "allow": true, "headers": {"x-route-to": "canary"} }
沒帶時              →  { "allow": true }   # 沒有 headers 這個 key，不是 headers: null
```

這個 **「有沒有這個 key」而不是「值是不是 null」** 的設計，之後會一直是這個架構判斷事情的方式——下游的機制是靠檢查某個欄位存不存在來決定要不要動作，不是檢查值是否為空。

（day08-1.png：帶 x-canary: true）

（day08-2.png：沒帶 x-canary）

## `opa` plugin：APISIX 跟 OPA 之間溝通的橋樑

APISIX 要透過 `opa` plugin 才能跟 OPA 對話。實際的設定放在 `enterprise-gitops-config` repo 的 `manifests/apps/demo-app/apisix-route-demo.yaml`——這份檔案裡 `opa` plugin 是跟 `traffic-split` plugin 一起宣告的（`traffic-split` 是之後才會講的內容，這裡先只看 `opa` 那段）：

```yaml
plugins:
  - name: opa
    config:
      _meta:
        priority: 2000
      host: "http://opa-service.enterprise-gitops.svc.cluster.local:8181"
      policy: "canary"          # 對應 Rego 的 package canary
      send_headers_upstream:
        - x-route-to            # 只有列在這裡的 key，才會被真的轉發進 request header
```

- `policy: "canary"` 對應 Rego 檔案裡的 `package canary`——這是 `opa` plugin 怎麼知道要問「哪一份」規則的依據。
- `send_headers_upstream` 決定 OPA 回傳的 `headers` 物件裡，哪些 key 會被真的寫進這筆 request 的 HTTP header。沒列出來的 key，就算 OPA 真的算出來了，也不會被轉發下去。

### `send_headers_upstream` 寫入 header 時，原本的 header 會怎樣？

**不會被清空，只有列在 `send_headers_upstream` 裡的那幾個 key 會被動到，其他 header 完全不受影響。** 這個細節官方 plugin 文件頁面沒有明講，查證的是 APISIX 原始碼 `apisix/plugins/opa.lua`：

```lua
for _, name in ipairs(conf.send_headers_upstream) do
    local value = headers[name]
    core.request.set_header(ctx, name, value)
end
```

- 這段代碼只會走訪 `send_headers_upstream` 清單裡列出的 header 名稱，一個一個呼叫 `core.request.set_header(ctx, name, value)` 去設定——沒被列在這個清單裡的其他 header，這段代碼完全不會碰。所以 **不是「先清空整個 request 的 header，再把 OPA 回傳的東西寫進去」**，是針對性地 **只設定指定的幾個 key**。
- **如果 request 本身已經帶有同名的 header**，`core.request.set_header` 是「設定/覆寫」語意，OPA 算出來的值會直接覆蓋掉原本那個 key 的值，不會兩個值並存，也不會變成一個 header 帶多個值。

## 參考資料

- [APISIX 官方文件 — opa plugin](https://apisix.apache.org/docs/apisix/plugins/opa/)：明講 `allow` 是「indispensable」（不可或缺）的欄位，用來決定 request 能不能通過 APISIX；`reason`／`headers`／`status_code` 則是選填，只有設定自訂回應時才會用到——這就是 `allow` 在 `opa` plugin 裡地位特殊的官方依據。
- [APISIX 原始碼 — apisix/plugins/opa.lua](https://github.com/apache/apisix/blob/master/apisix/plugins/opa.lua)：`send_headers_upstream` 逐一設定 header、不清空其他欄位的實作邏輯，官方文件頁面沒有寫到這麼細，這裡是查證的第一手依據。
