# Day 04：用 ApisixRoute CRD 建立 APISIX 的第一條路由規則

## 今天的工作

1. 用 `ApisixRoute` CRD 建立 APISIX 的第一條路由規則

## 建立第一條路由規則

昨天我們把 APISIX 佈署起來了，原代碼裡也包含在雲平台建立一個 load balancer。  
今天我們要替 APISIX 建立第一條路由規則。

### APISIX 的 2 個入口

APISIX 有兩個不同用途的「門」：

**一般流量的門**：處理 Client 打進來的請求，依路由規則轉發到對應的 backend。

**`Admin API`（設定 APISIX 本身的門）**：新增一條路由、改一個插件設定，都得對這個門發 HTTP 請求（`GET`／`PUT`／`DELETE`）才能做到，不是直接改 APISIX 內部的路由表。

### APISIX 的 2 個重要 CRD

**`ApisixRoute`**：`ApisixRoute` 讓我們可以用 YAML 寫出「符合什麼條件的 request，要轉發去哪個 backend」的路由規則。昨天提到的 **Ingress Controller** 會把 `ApisixRoute` 裡的路由規則轉換成對 Admin API 的 HTTP 請求。

**`GatewayProxy`**：它告訴 Ingress Controller 兩件事

- Admin API 的位址（Service 名稱、port），讓 Ingress Controller 知道向哪個 API 發送請求。
- 認證用的金鑰

這 2 個 CRD 的好處是：Admin API 的位址、金鑰可以獨立更新／輪替，不需要重新部署 Ingress Controller 本身。

**Ingress Controller 監聽到 `ApisixRoute` 之後的工作流程**：

1. Ingress Controller 持續 watch K8s API，偵測到新增或變更的 `ApisixRoute` CRD
2. 查對應的 `GatewayProxy`，取得這個叢集的 Admin API 位址跟金鑰
3. 用這組連線資訊呼叫 Admin API，把 `ApisixRoute` 宣告的內容寫進 APISIX 的路由表

APISIX Helm Chart 有一個 `gatewayProxy.createDefault: true` 開關（在 `manifests/infrastructure/apisix/values.yaml`），啟用後會自動幫我們產生一份預設的 `GatewayProxy`，指向這個叢集裡的 APISIX Admin API，不用手動寫。

### 用 ApisixRoute 寫路由

`ApisixRoute` 寫法很直覺：告訴 APISIX **「符合哪些條件的 request，下一跳（next hop）要送去哪個 Service」**。我們來寫第一條路由：

```
如果 request 訪問的網域是 demo-app.example.com、路徑符合 /api/*、方法是 GET 或 POST，就全部送去 demo-app-stable 這個服務（port 80）。
```

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: demo-app-route
  namespace: enterprise-gitops
spec:
  ingressClassName: apisix
  http:
    - name: demo-app-stable-rule
      match:
        hosts:
          - demo-app.example.com
        paths:
          - /api/*
        methods:
          - GET
          - POST
      backends:
        - serviceName: demo-app-stable
          servicePort: 80
          weight: 100
```

> 原代碼放在 `config` repo 的 `manifests/apps/demo-app/apisix-route-demo.yaml`

幾個關鍵欄位：

- `ingressClassName: apisix` — 一定要寫，缺了這個，APISIX Controller 不會認領這條路由，K8s 不會報錯，但 Admin API 的路由表永遠是空的。
- `match` — 符合哪些條件才算命中這條路由：host、path、method
- `backends` — 命中之後送去哪個 Service，`weight` 決定分流比例（今天全部給 100，還沒有金絲雀分流機制）

### http 區塊

`spec.http` 是一個 **陣列**，每個元素都是一條獨立的路由規則——一個 `ApisixRoute` CR 可以裝好幾條規則。每條規則自己有 `name`、`match`、`backends`，彼此獨立比對，不互相影響。

如果哪天有兩條規則同時符合同一個 request（例如 path 有重疊），APISIX 用 `priority` 這個欄位決定誰優先——數字越大優先權越高。這個專案目前只有一條規則，用不到這個欄位。

### match 區塊

```yaml
match:
  hosts:
    - demo-app.example.com
  paths:
    - /api/*
  methods:
    - GET
    - POST
```

判斷邏輯分兩層：

- **同一個欄位裡列多個值，是「或」（OR）**：`methods` 列了 GET 和 POST，這個 request 是 GET **或** POST 就算命中，不用兩個都符合。
- **不同欄位之間，是「且」（AND）**：host 要對、path 要對、method 也要對，三個條件同時成立才算命中——只符合 path、host 不對，一樣不會命中。

三個欄位是同時判斷的，沒有先後順序，不是先比對 host 再比對 path 這種循序邏輯。

### backends 區塊

```yaml
backends:
  - serviceName: demo-app-stable
    servicePort: 80
    weight: 100
```

`backends` 也是一個陣列，決定「符合上面 match 條件的 request，要送去哪個（或哪幾個）Service」。`weight` 決定多個 backend 之間怎麼分流——現在只有一個 backend、weight 100，代表 100% 都送去 `demo-app-stable`。

這個欄位本身確實有能力列多個 backend、依權重拆流量，但 **這個專案不會用它做金絲雀分流**：後面的金絲雀邏輯是靠 `traffic-split` 這個獨立的 plugin 來做（Day09 開始），這裡的 `backends` 從今天到專案結束都只會有 `demo-app-stable` 這一個、weight 固定 100，不會再被改動。

## 參考資料

- APISIX Ingress Controller 官方文件，`ApisixRoute`／`GatewayProxy` 等 CRD 的定義：[APISIX Ingress Controller Resources](https://apisix.apache.org/docs/ingress-controller/concepts/resources/)
