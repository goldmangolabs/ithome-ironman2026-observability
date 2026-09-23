# Day 09：接上第二個 Plugin——traffic-split

## 今天的工作

1. 介紹 APISIX 的 plugin pipeline 的運作方式
2. 認識第二個 plugin：`traffic-split`
3. 看懂 `opa` 跟 `traffic-split` 這兩個 plugin 怎麼合作，把金絲雀分流真正做出來

Day08 我們認識了 OPA、接上了 `opa` plugin，但那時候只做到「問決策」——真正把流量轉去 Canary 的動作，還沒有任何 plugin 在做。今天要接上負責「執行轉發」的 `traffic-split` plugin。

## Plugin Pipeline 怎麼運作

一條 `ApisixRoute` 底下可以有多個 plugin。它們會依照 `_meta.priority` 這個數字，由大到小依序組成一條處理 request 的 pipeline。

這裡有一個容易誤解的地方：**plugin 之間不會直接呼叫或對話**，它們是各自獨立執行的。一個 plugin 執行完之後，如果對這筆 request 做了任何修改（例如加了一個 HTTP header），下一個 plugin 執行時看到的就是「已經被改過的 request」——它不知道也不需要知道這個改動是誰做的，它只是照著現在的 request 內容執行自己的邏輯。這條 pipeline 唯一的傳遞媒介，就是 request 本身的狀態，不是 plugin 之間的直接溝通。

## 認識第二個 plugin：`traffic-split`

`traffic-split` 的工作很單純：依照設定的條件，把原本要送去路由預設 `backends` 的流量，**覆寫** 改送去別的後端。它自己不會判斷「這個 request 該不該走金絲雀」，只負責檢查條件、決定要不要覆寫。

真實設定接在 Day08 那份檔案的 `opa` plugin 後面（`manifests/apps/demo-app/apisix-route-demo.yaml`）：

```yaml
- name: traffic-split
  config:
    _meta:
      priority: 966
    rules:
      - match:
          - vars:
              - ["http_x-route-to", "==", "canary"]
        weighted_upstreams:
          - upstream:
              name: canary-upstream
              type: roundrobin
              nodes:
                "demo-app-canary.enterprise-gitops.svc.cluster.local:80": 1
            weight: 100
```

拆解這幾個區塊：

- **`rules`**：一個陣列，裡面可以放 **不只一條** 規則，每條規則各自有自己的 `match` 跟 `weighted_upstreams`。APISIX 會照陣列順序依序檢查，**第一條 match 成立的規則生效，後面的規則不會再檢查**；如果所有規則都不成立，就 fallback 回路由層級預設的 `backends`。

- **`match`**：這條規則自己的過濾條件，寫法跟 `ApisixRoute` 頂層的 `match` 很像，一樣是陣列，裡面可以放多組條件，彼此是「OR」的關係——只要其中一組成立，這條規則就算 match。我們這裡只用了 `vars` 這一種條件（比對 `http_x-route-to` 這個變數）。

- **`weighted_upstreams`**：這條規則 match 成立後，要送去的候選後端清單，也是陣列，可以放 **不只一個** 目的地。這是 `traffic-split` 真正強大的地方：不只能「全部導去 A」，還能在多個候選後端之間依權重切分流量（例如 A 80%、B 20%）。

- **`upstream`**（`weighted_upstreams` 底下每一項的內容）：`name` 是這組後端的識別名稱；`type` 是負載平衡演算法（`roundrobin` 輪詢、`chash` 一致性雜湊、`ewma`、`least_conn` 最少連線數，我們這裡用最基本的 `roundrobin`）；`nodes` 是實際的後端位址清單，可以列多個節點、各自帶權重。

- **`weight`**（跟 `upstream`同一層、平行的欄位）：當 `weighted_upstreams` 裡有多個候選後端時，這個數字決定「這個後端要分到多少比例的流量」——是流量切分的權重，跟上面 `nodes` 裡的權重是不同層級的東西（`nodes` 分的是同一組後端內部多個節點之間的流量，`weight` 分的是多組候選後端彼此之間的流量）。

## `opa` 跟 `traffic-split` 怎麼合作

它們排好順序，像工廠的生產線一樣，讓 request 的 HTTP header 通過自己的區域並加工，或者根據 HTTP header 採取相應的行動：

1. `opa`（priority 2000）先執行。收到 OPA 回傳的判斷結果後，透過 `send_headers_upstream` 把 `x-route-to: canary` 這個 header **寫進** 這筆 request。
2. `traffic-split`（priority 966）後執行。它讀的是這筆 request **現在** 的 header，語法是 `http_x-route-to`。這裡有個容易踩到的地方：教科書等標準 nginx 慣例（例如 `$http_x_forwarded_for`）會把 header 名稱裡的連字號轉成底線，但 APISIX 的 `vars` 條件比對語法是**例外**，會保留原始連字號——寫成 `http_x_route_to` 永遠比對不到值，一定要寫成 `http_x-route-to`。對 `traffic-split` 來說，這個 header 是誰寫的、什麼時候寫的都不重要，它只看「現在有沒有這個 header、值是什麼」。
3. 條件成立（`http_x-route-to == "canary"`），`traffic-split` 就用 `weighted_upstreams` 裡設定的候選後端，覆寫掉路由層級預設的 `backends`，把這筆流量轉去 `demo-app-canary`；條件不成立（沒有這個 header，或值不是 `"canary"`），`traffic-split` 什麼都不做，流量照 ApisixRoute 原本的路由 `backends` 設定，送去預設的 `demo-app-stable`。

## 參考資料

- [APISIX 官方文件 — Plugin](https://apisix.apache.org/docs/apisix/terminology/plugin/)：Plugin pipeline 與 `_meta.priority` 執行順序的官方說明。
- [APISIX 官方文件 — traffic-split plugin](https://apisix.apache.org/docs/apisix/plugins/traffic-split/)：`rules`／`match`／`weighted_upstreams` 完整 schema 說明，包含負載平衡演算法選項與規則比對順序。
