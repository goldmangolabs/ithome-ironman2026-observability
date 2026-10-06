# Day 22：補齊 APISIX 網關監控 — Upstream 延遲

## 今天的工作

1. 建立 Dashboard，監控 APISIX 與 Upstream 的 P95 Latency
2. 開流量產生器，建立基線流量
3. 對 Pod 注入故障，驗證面板
4. 復原，驗收 Pod 恢復健康

Day17 就已經啟用 APISIX 的 `prometheus` 插件，把 `apisix_http_latency`（延遲分佈）送進 Prometheus，今天用 Dashboard 觀察這個指標。它可以初步了解 APISIX 和後端服務的溝通延遲情況。

## P95 Upstream Latency 的工作邏輯

`apisix_http_latency{type="upstream"}` 量的是 **APISIX 自己作為呼叫方，從它把請求送進 upstream（也就是這個請求實際被路由到的那個後端）開始，一直到收到完整回應為止**，這一段花的時間，包含了 **網路延遲** 和 **後端服務處理時間**。如果連線真的變慢、斷線或封包遺失，APISIX 會被卡在等待 upstream 回應，這段等待時間會明顯拉高延遲數字，甚至一路頂到逾時。加上 Day18 建好的 Trace（可以看清楚一次請求的時間到底花在哪一段、打到哪個後端），就足以回答「APISIX 到後端現在通不通」這個問題。

### 用 Pod IP 分線的原因和局限

這張面板的 PromQL 用 `by (le, node)` 分組，圖例直接印出每個後端 IP 各自的 P95，目的是可以用 Pod IP 定位出問題的 Pod。可以用這個方法觀察 APISIX 和 Stable / Canary 之間的延遲。

局限：**這個設計拿來做 PoC 沒問題，正式環境不容易這樣觀察。** Pod IP 不是身分，是位址。每次重啟就換一個 IP，舊 IP 的線在圖上會斷掉變孤兒，新 IP 又冒出一條全新的線。這個專案 Stable、Canary 各只有 1 個 replica，重啟也不頻繁，圖例上兩三個 IP 還能人眼對照著看；正式環境動輒數十個 workload 和多個 replica，用 IP 分線很快就會變成一堆看不出意義的雜訊，人眼完全對不上哪條線是哪個版本。要在正式環境做到這件事，得另外接一層對應（例如拿 `kube-state-metrics` 的 Pod label 去 join `node` 這個 IP），考慮到時間局限性，這個沒有做 XD。（這是一個很好的研究方向！可以放到下個 POC 裡！）

## 建立 APISIX Upstream Health：P95 Upstream Latency 面板

```yaml
# manifests/infra/observability/dashboards/apisix-upstream-health.yaml
{
  "uid": "apisix-upstream-health-day22",
  "title": "APISIX Upstream Health",
  "panels": [
    {
      "id": 1,
      "title": "P95 Upstream Latency（依後端 Pod IP 分線）",
      "targets": [{
        "expr": "histogram_quantile(0.95, sum(rate(apisix_http_latency_bucket{type=\"upstream\"}[1m])) by (le, node))",
        "legendFormat": "{{node}}"
      }]
    }
  ]
}
```

## 開流量產生器，建立基線流量

我們替 Stable 跟 Canary 分別建立基線流量，才能從圖上觀察：

```bash
APISIX_IP=$(kubectl get svc apisix-gateway-gateway -n <namespace> -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Stable 流量（不帶 x-canary，落在預設路徑）
while true; do
  curl -s -o /dev/null -H "Host: demo-app.example.com" "http://${APISIX_IP}/api/hello"
  sleep 0.5
done &

# Canary 流量（帶 x-canary: true，導向 Canary）
while true; do
  curl -s -o /dev/null -H "Host: demo-app.example.com" -H "x-canary: true" "http://${APISIX_IP}/api/hello"
  sleep 0.5
done &
```

## 故障注入驗證：測試延遲面板

我們要怎麼測：

1. 用 K8s 的 Ephemeral Container 的機制，塞一個除錯工具 `chaos-debugger` container 進入 Canary Pod，跟 Canary 共用同一張網卡。這個機制不用重啟 Pod。
2. 用 `chaos-debugger` 的 `tc` 工具，讓 Canary 送出去的每個封包都先延遲 500ms 再送出。
3. APISIX 因此得多等 500ms 才收到 Canary 的回應。
4. 觀察 APISIX Upstream Health 看板，確認 Canary 那條線真的跳，Stable 那條線沒事。

實際操作用的是我們自己寫的 `scripts/inject-chaos.sh`：

```bash
./scripts/inject-chaos.sh --target demo-app-canary --action delay --ms 500
```

注入後，觀察 Dashboard：APISIX Upstream Health。應該會看到：Canary 那個 IP 的線出現明顯尖峰，Stable 那個 IP 的線沒什麼變化。

## 復原，驗收

### 停掉測試流量

先停掉背景跑的兩個流量產生器。它們是用 `&` 丟到背景的，不會自己停。在同一個 shell 裡用 `kill` 停掉剛才那兩個背景工作：

```bash
kill %1 %2
```

### 清除 "延遲機制"

再清除故障、確認 Pod 恢復健康：

1. 停止 `tc` 對 Canary Pod 的封包的延遲發送
2. `chaos-debugger` container 要等到下次重建新 Pod 後才會被剔除。

```bash
./scripts/inject-chaos.sh --target demo-app-canary --action clear

kubectl wait --for=condition=Ready pod -l app=demo-app,track=canary,version=v2-canary \
  -n <namespace> --timeout=60s
```

## 參考資料

- [Apache APISIX — prometheus Plugin](https://apisix.apache.org/docs/apisix/plugins/prometheus/)：`apisix_http_latency{type="upstream"}` 這個指標量的是 APISIX 到 upstream 這一段延遲的官方定義依據。
- [apache/apisix 原始碼 — `apisix/utils/log-util.lua`](https://github.com/apache/apisix/blob/master/apisix/utils/log-util.lua)：`upstream_latency` 實際讀的是 Nginx 內建變數 `$upstream_response_time`，是「這個延遲混合了網路傳輸時間與後端處理時間、無法拆分」這個說法的原始碼依據。
