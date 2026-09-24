# Day 10：QA 怎麼用一個 Header 精準打到 Canary

## 今天的工作

1. 介紹「QA 要能精準存取 Canary」這個需求，以及做法
2. 看 `opa` plugin 送給 OPA service 的請求裡，HTTP header 實際長什麼樣子
3. 看 OPA 怎麼被設定（Rego 規則），才能針對這個請求回應「導向 Canary」

Day08、Day09 我們把 `opa`、`traffic-split` 兩個 plugin 都接上了，管線已經搭好。今天用一個具體的使用情境，把這條管線從頭到尾走一遍：QA 人員要怎麼「指定」自己一定會被導向 Canary。

## 需求：QA 要能精準訪問 Canary，不能靠機率

QA 人員做測試的時候，需要 **每次都能確定訪問 Canary**，所以我們要設計一條 **可以被明確指定** 的路徑。

### 做法

QA 發出請求時，帶上一個 HTTP header `x-canary: true`。這個 header 是這個專案自己定義的——只要 `opa` plugin 把它送進 OPA service、Rego 規則認得這個 header，就能返回特定的 JSON 內容，達成「這個 request 一定被導向 Canary」的效果。

跟 Day05、Day07 一樣，實際送出這個請求用的是 `curl -H`：

```bash
curl -H "Host: demo-app.example.com" \
     -H "x-canary: true" \
     http://<APISIX_EXTERNAL_IP>/api/hello
```

## `opa` plugin 送給 OPA 的請求，header 長什麼樣

`opa` plugin 會把整筆 request 的特徵包成 JSON（也就是 Day08 提過的 `input`），送去 OPA service。當 QA 帶著 `x-canary: true` 打過來，實際送出的 `input` 長這樣：

```
{
  "input": {
    "type": "http",
    "request": {
      "scheme": "http",
      "path": "/api/hello",
      "headers": {                # http request 的 headers 底下全部的 header
        "host": "demo-app.example.com",
        "x-canary": "true",          # 這個就是用來影響 OPA service 判斷的關鍵！！
        "user-agent": "curl/...",
        "accept": "*/*"
      },
      "query": {},
      "method": "GET",
      "host": "demo-app.example.com"
    }
  }
}
```

## OPA service 怎麼被設定，才能回應「導向 Canary」

Day08 已經看過完整的 Rego 規則，這裡把它對上今天這個具體的 `input`，走一次真正的判斷過程：

```rego
headers = {
    "x-route-to": "canary"
} {
    input.request.headers["x-canary"] == "true"
}
```

OPA 收到上面那包 `input` 後，去查 `input.request.headers["x-canary"]`——查得到，值是 `"true"`，條件成立，`headers` 這個變數被賦值，OPA service 回傳：

```json
{ "allow": true, "headers": {"x-route-to": "canary"} }
```

如果今天 QA 忘了帶 `x-canary` 這個 header，`input.request.headers` 裡就不會有這個 key，`input.request.headers["x-canary"]` 查到的是 undefined，跟字串 `"true"` 永遠不相等，條件不成立，`headers` 整個不存在，OPA 只會回傳 `{ "allow": true }`。

## 接下來，交給 `traffic-split` plugin

但 OPA 回傳 JSON，還不等於這筆請求真的被導向 Canary——中間還要接回 Day09 講過的那條管線：

- `opa` plugin 收到 `{"headers": {"x-route-to": "canary"}}` 之後，透過 `send_headers_upstream` 把 `x-route-to: canary` 真的寫進這筆 request 的 header。

- 接著換 `traffic-split` 執行，它讀 `http_x-route-to` 這個變數，比對到 `"canary"`，才會用 `weighted_upstreams` 裡設定的候選後端，把請求覆寫轉發到 `demo-app-canary`。

QA 忘了帶 header 的情境也是同一條路徑走一次：

- OPA 回傳的 JSON 裡沒有 `headers` 這個 key，`opa` plugin 沒有東西可以寫進去，`x-route-to` 這個 header 從頭到尾不存在。

- `traffic-split` 讀到的 `http_x-route-to` 是空的，條件 `== "canary"` 不成立，這條規則不生效，請求就照 `ApisixRoute` 頂層 `backends` 的預設值，落回 `demo-app-stable`。

## 提醒

- `x-canary` 這個 header 名稱是這個專案自己定義的，OPA、APISIX 默認不會對這個名字做任何特殊處理。真正讓它產生效果的，是 Rego 規則裡明確寫了 `input.request.headers["x-canary"]` 這一行去讀它。換一個 header 名稱一樣可以運作，只要 Rego 規則也跟著改。

## 參考資料

- [APISIX 原始碼 — apisix/plugins/opa/helper.lua](https://github.com/apache/apisix/blob/master/apisix/plugins/opa/helper.lua)：`headers = core.request.headers(ctx)`，證明送給 OPA 的 `input.request.headers` 是完整、未過濾的請求 header 表。官方 plugin 文件頁面只給了範例 JSON，沒有明講「是不是全部」，這裡是查證的原代碼依據。
