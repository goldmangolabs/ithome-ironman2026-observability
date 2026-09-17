# Day 03：部署 APISIX

## 今天的工作

1. 介紹 APISIX
2. LoadBalancer 如何與 APISIX 一起工作
3. 佈署 APISIX

## APISIX 是什麼？

APISIX 是一個雲原生 API 閘道器，底層基於 Nginx/OpenResty + Lua，由 Apache 軟體基金會維護。它透過 Admin API 管理 Route、Upstream、SSL、Consumer 等設定資源，官方稱這些設定跟 Plugin 的更新 **「不需要重啟」**（Hot Updates And Hot Plugins）。功能上涵蓋動態路由、負載均衡、限流熔斷、身分驗證，支援多種插件。在 K8s 場景下也有對應的 Ingress Controller。

在我們的架構裡，APISIX 扮演的角色是 **HTTP requests 分流**。所有進入 `demo-app` 的請求，都會先經過 APISIX，再由它根據路由規則決定把流量導向 Stable 版本（v1）還是 Canary 版本（v2）。

APISIX 的部署包含兩個組件：

| 組件 | 職責 |
|------|------|
| **Data Plane（Gateway）** | 實際轉發流量，根據記憶體中的路由表做決策 |
| **Ingress Controller** | 監聽 K8s 的 `ApisixRoute` CRD，將其轉換為 Etcd 中的路由規則 |

兩者協作的方式是：Ingress Controller 把宣告式的路由規則寫進 Etcd，Data Plane 透過 Watch 機制即時取得更新。**如果只部署 Data Plane 而沒有啟用 Ingress Controller，Day 04 宣告的 `ApisixRoute` 不會生效。**

### 為什麼不用 Istio 或者 Nginx？

這 3 個工具都能分流 request。選擇 APISIX 主要的原因是：

1. APISIX 是一個基於 Nginx 開發出來的工具，其實，我們就是在利用 Nginx 強大的 HTTP request 管理能力，把 Nginx 改得 "更適合雲原生環境的 K8s 架構"，例如把它分成 "control plane" 和 "data plane" 這種工作架構。這讓我們在思考 APISIX 的設計的時候，和 "雲原生" 的其他工具保持一致性。
2. Istio 要設定 side car 網格，考慮到本專案的需求，這個設定 **較為繁鎖**，而 APISIX 設定相對單純。

---

## 2. Load Balancer 與 APISIX 的工作邏輯

本專案使用的 GCP Load Balancer 是 layer 4，不會去解析 HTTP；APISIX 收到封包後，它會解析 HTTP request 來決定路由，並且把 request 傳送到路由的 backend。

### GCP L4 LB 的工作邏輯

1. **Health Check 篩選對象**：GCP 對每個 Node 做健康檢查時，打的是 kube-proxy（或 cilium-agent）負責回應的一個獨立健康檢查 port——只要這個 Node 上有 kube-proxy 在正常運作，就會被列入可接收流量的名單。
2. **5-tuple hash 選 Node**：外部封包進來時，LB 依據來源 IP/Port、目的 IP/Port、協定這 5 個欄位算 hash，從健康的 Node 名單裡挑一個，把封包**原封不動**（不解封裝、不看 HTTP 內容）轉發過去——這就是「passthrough」的意思，純網路層轉發，不是代理伺服器。
3. **封包送達 Node 的網卡**，但這個 Node 不一定真的跑著 APISIX Pod。

### 封包進 Node 之後，怎麼到 APISIX Pod

1. 封包到 Node 上時，目的地是 `Node-IP:NodePort`。這個 Node 的 kube-proxy 攔下這個封包，查自己維護的 Service Endpoints 清單（也就是這個 Service 底下目前所有健康的 Pod IP），挑一個，做 **DNAT**：把目的地從 `Node-IP:NodePort` 改寫成 `Pod-IP:ContainerPort`，再送進那個 Pod。
2. 如果挑到的 Pod 剛好不在原本收到封包的那個 Node 上，還會多一跳轉發到別的 Node——這一跳預設會做 SNAT，代表 APISIX 看到的來源 IP 會變成 Node IP，不是真正的 client IP（除非設定 `externalTrafficPolicy: Local` 犧牲跨 Node 轉發能力去換保留真實來源 IP）。

### APISIX 接手之後

封包送到 APISIX Pod 監聽的 Port（例如 9080）時，才第一次被解析成 HTTP——APISIX 的 Worker 開始看 Method、Path、Header、Cookie，比對 Etcd 裡同步好的 `ApisixRoute` 規則，呼叫 `opa` plugin（Day08 內容） 問 OPA 該不該走金絲雀，再由 `traffic-split` plugin（Day09 內容）決定實際打去哪個 upstream。

## 佈署 APISIX

這個專案的 APISIX 是用 git commit 觸發部署：把設定檔推進 Config Repo，ArgoCD 偵測到變更後自動同步、幫你執行 `helm install`。

需要的檔案，全部放在 `config` 這個 repo：

| 檔案 | 路徑 | 內容 |
| --- | --- | --- |
| AppProject 更新 | `manifests/argocd-apps/appproject.yaml` | 把 APISIX 的 Helm Repo（`https://charts.apiseven.com`）加進白名單，不加的話 ArgoCD 會拒絕同步 |
| APISIX Helm Values | `manifests/infrastructure/apisix/values.yaml` | APISIX 實際的設定（副本數、怎麼連 Etcd、開哪些 plugin） |
| APISIX ArgoCD Application | `manifests/argocd-apps/apisix-app.yaml` | 告訴 ArgoCD 要監控哪個 Helm Chart、套用哪份 values 檔 |

## 提醒

- 這裡的 Etcd 設定了 `replicaCount: 3`，但為了省錢，把 `podAntiAffinityPreset` 從正式環境該用的 `hard` 降成 `soft`，3 個 Etcd Pod 並沒有強置分散在 3 個 node 裡，在實驗初期可以開一個 work node 就足夠。

## 參考資料

- 官方文件：[Configure a Service to use a network load balancer](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/service-load-balancer)

- APISIX 官方 README「Full Dynamic」章節，[apache/apisix README](https://github.com/apache/apisix/blob/master/README.md)

- APISIX 官方 Admin API 文件，列出 Route、Service、Upstream、SSL、Consumer 等可透過 API 管理的資源類型——[Admin API](https://apisix.apache.org/docs/apisix/admin-api/)
