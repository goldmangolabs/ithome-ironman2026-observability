# Day 11：內部員工用 Cookie 存取 Canary

## 今天的工作

1. 介紹「內部員工要能存取 Canary，但不能靠 Header」這個需求，以及做法
2. 看 `opa` plugin 送給 OPA service 的請求裡，帶 Cookie 的 header 長什麼樣子
3. 看 OPA 怎麼被改設定（Rego OR 邏輯），才能同時認得 Header 跟 Cookie 兩種存取方式

Day10 我們讓 QA 能用 `x-canary` 這個 Header 精準打到 Canary。今天要多開一條路——內部員工透過瀏覽器登入系統拿到 Cookie，不會手動塞 Header，所以要讓 OPA 多認得一種存取方式。

## 需求：內部員工不會手動改 Header，只有 Cookie

QA 人員可以用 Postman 或腳本手動帶 Header，但內部員工平常就是打開 Chrome 瀏覽，不會、也不應該要求他們手動加 Header 才能測試 Canary。企業級場景裡，員工的身份通常是透過登入系統寫進瀏覽器的 Cookie——所以我們要讓 OPA 多認一種條件：只要 Cookie 裡帶有 `canary=true`，一樣導向 Canary。

### 做法

跟 Day10 一樣用 `curl`，差別是這次用 `-b` 帶 Cookie，不是 `-H` 帶 Header：

```bash
curl -H "Host: demo-app.example.com" \
     -b "session_id=a1b2c3d4; theme=dark; canary=true" \
     http://<APISIX_EXTERNAL_IP>/api/hello
```

> 通常瀏覽器送出的 Cookie 都會包含很多其他跟這次測試無關的內容（登入 session、介面偏好設定……），不會只有單獨一個 `canary=true`。這裡刻意在 `-b` 裡塞幾個不相干的 cookie，模仿真實情況。

## `opa` plugin 送給 OPA 的請求，帶 Cookie 的 header 長什麼樣

Cookie 在 HTTP 裡本來就是一個叫 `Cookie` 的 header，值是所有 cookie 用分號串起來的字串。瀏覽器真實送出的請求，`cookie` 這個 key 通常不會只有一個值，會混著其他站台自己的 cookie：

```json
{
  "input": {
    "type": "http",
    "request": {
      "scheme": "http",
      "path": "/api/hello",
      "headers": {
        "host": "demo-app.example.com",
        "cookie": "session_id=a1b2c3d4; theme=dark; canary=true",
        "user-agent": "Mozilla/5.0 ...",
        "accept": "text/html"
      },
      "query": {},
      "method": "GET",
      "host": "demo-app.example.com"
    }
  }
}
```

## OPA 怎麼被改設定，才能同時認得 Header 跟 Cookie

把 Day08 那條只判斷 Header 的規則，改成一個叫 `is_canary` 的規則，定義兩次：

```rego
is_canary {
    input.request.headers["x-canary"] == "true"
}

is_canary {
    contains(input.request.headers["cookie"], "canary=true")
}

headers = {
    "x-route-to": "canary"
} {
    is_canary
}
```

- **同一個規則名稱定義兩次，就是 Rego 表達 OR 邏輯的方式**——官方文件的說法是「incrementally defined rule」，兩個定義各自獨立判斷，只要任一個成立，`is_canary` 就成立，等同 `<規則1> OR <規則2>`。

- **`contains(haystack, needle)`** 是 Rego 內建的字串函式，檢查 `haystack` 裡有沒有包含 `needle` 這個子字串，回傳布林值。對照上面的例子：`contains("session_id=a1b2c3d4; theme=dark; canary=true", "canary=true")` 回傳 `true`，因為那串 cookie 裡確實包含 `"canary=true"` 這段文字。

- `input.request.headers["cookie"]` 拿到的是 **整串** `"session_id=a1b2c3d4; theme=dark; canary=true"`，不是單獨一個 `canary=true`——這就是為什麼 Rego 判斷式不能用 `==` 精確比對，只能用「這串裡面有沒有包含 `canary=true`」來判斷。

- `headers` 這條規則本身沒有變複雜——它現在只依賴一個判斷式 `is_canary`，不管 `is_canary` 是被 Header 條件還是 Cookie 條件滿足的，`headers` 的行為完全一樣。

## 接下來，交給 `traffic-split` plugin

管線的下半段跟 Day10 完全一樣，沒有任何改變：`opa` plugin 透過 `send_headers_upstream` 把 `x-route-to: canary` 寫進 request，`traffic-split` 讀 `http_x-route-to`，match 到就覆寫轉發到 `demo-app-canary`。今天唯一改變的，是 OPA **怎麼判斷** 要不要寫這個值，`opa` plugin 跟 `traffic-split` 之後的行為完全沒動。

## 提醒

- 如果 QA 跟員工的條件都不成立，`is_canary` 兩個定義都不成立，整個規則是 undefined，跟 Day08、Day10 講過的一樣，`headers` 完全不存在，`traffic-split` 讀不到 `x-route-to`，落回 `demo-app-stable`。
- 真實環境會用簽章 token，這裡簡化成明文字串方便教學。

## 參考資料

- [OPA 官方文件 — Policy Language：Incremental rule definition](https://www.openpolicyagent.org/docs/latest/policy-language/)：「An incrementally defined rule can be intuitively understood as `<rule-1> OR <rule-2> OR ... OR <rule-N>`」——同名規則定義多次＝OR 邏輯的官方依據。
- [OPA 官方文件 — Built-in Functions：`contains`](https://www.openpolicyagent.org/docs/policy-reference/builtins)：`contains(haystack, needle)` 的官方簽章與說明。
- [curl 官方文件 — manpage：`-b`／`--cookie`](https://curl.se/docs/manpage.html)：「pass the exact data to send to the HTTP server in the Cookie header」——證明 `-b` 只是 `Cookie:` header 的方便寫法，跟 `-H` 送出去的東西在協定層面完全一樣，這是 `cookie` 會跟其他 header 一起出現在同一個 `headers` 物件裡的依據。
