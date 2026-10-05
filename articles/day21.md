# Day 21：金絲雀核心決策看板 — Canary Traffic Split

## 今天的工作

1. 用 git 管理 Dashboard，實現 Dashboard as Code
2. 用 RED Method 設計一張「一眼看懂該不該退版」的決策看板，搭配 Day14 的「一鍵停止 Canary」
3. `$version` 動態變數，讓每個面板同時比較 Stable／Canary，不用寫兩份 PromQL
4. 驗證：送測試流量，確認每個面板都有真實數據

## Dashboard as Code

把 Dashboard 匯出成 JSON，包進 K8s 的 `ConfigMap`，我們就可以用 git 管理 Dashboard 了！
不過，建議先用 Grafana GUI 製作出來 Dashboard，再匯出 JSON 格式。因為直接寫 JSON 格式的 Dashboard 很不直覺。

```yaml
# manifests/infra/observability/dashboards/canary-traffic-split.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-canary-traffic-split
  namespace: observability
  labels:
    grafana_dashboard: "1"
data:
  canary-traffic-split.json: |
    { "uid": "canary-traffic-split-day21", "title": "Core: Canary Traffic Split", ... }
```

## 設計：RED Method

這張看板回答一個問題：**現在該不該退版？**

**RED Method** 是 Tom Wilkie 在 2015 年提出的微服務監控框架，只看三件事：**Rate（速率）、Errors（錯誤）、Duration（耗時）**。這三個訊號，回答的是「使用者實際感受到的服務好不好」。

Day17 提過，APISIX 本身的指標沒辦法用來區分 Stable/Canary。原因是：`traffic-split` 插件是在同一條 Route 裡，靠 header 權重選上游，APISIX 自己的 Prometheus 指標並不會記錄這次請求選中了哪個 upstream，沒有任何標籤可以拿來分版本。所以這裡所有面板改用應用層的 `service_version` 標籤（`v1`／`v2-canary`），直接查 OTel Java Agent 吐出的 `http_server_request_duration_seconds_*` 指標。

看板頂端放一張 RED Method 彙整表格：

```json
{
  "title": "🔴 RED Method Summary — 看到紅色立即執行 Kill Switch",
  "targets": [
    { "expr": "(sum(rate(http_server_request_duration_seconds_count{http_response_status_code=~\"5..\", service_name=\"demo-app\", service_version=\"v2-canary\"}[2m])) or on() vector(0)) / sum(rate(http_server_request_duration_seconds_count{service_name=\"demo-app\", service_version=\"v2-canary\"}[2m]))" },
    { "expr": "histogram_quantile(0.95, sum(rate(http_server_request_duration_seconds_bucket{service_name=\"demo-app\", service_version=\"v2-canary\"}[2m])) by (le))" }
  ]
}
```

Canary Error Rate 超過 1%，或 P95 Latency 超過 200ms，數字直接變紅——不用解讀趨勢，看到紅色就執行 **Day14 的 Kill Switch。**

## `$version` 動態變數：一份 PromQL 兩個版本

其餘的面板（流量分配比例、QPS 對比、P95 延遲對比、5xx 錯誤率對比、JVM Heap 趨勢）不寫兩份 PromQL，而是靠 Grafana 的模板變數，一份查詢同時畫出 Stable、Canary 兩條線：

```json
{
  "name": "version",
  "type": "custom",
  "multi": true,
  "query": "v1,v2-canary",
  "current": { "value": ["v1", "v2-canary"] }
}
```

面板裡的 PromQL 用 `${version}` 取代寫死的版本字串：

```promql
sum(rate(http_server_request_duration_seconds_count{service_name="demo-app", service_version=~"${version}"}[1m])) by (service_version)
```

以後如果加了 `preview` 這種第三個環境，只要在變數選項多加一個字，整張看板就能無痛支援三條線比對。

## 驗證：送測試流量，確認每個面板都有真實數據

驗證策略：先確認 Dashboard 真的被 Grafana 載入（查 `uid=canary-traffic-split-day21`），再送一批 Stable／Canary 測試流量，最後逐一對照面板用到的 PromQL，確認全部查得到非空資料——不是只看 Dashboard 存不存在，是看它畫不畫得出東西。

```bash
# 取得 Grafana admin 密碼並 port-forward
kubectl get secret kube-prometheus-stack-grafana -n observability \
  -o jsonpath='{.data.admin-password}' | base64 -d
kubectl port-forward svc/kube-prometheus-stack-grafana -n observability 3000:80

# 確認 Dashboard 已載入
curl -s -u admin:<密碼> "http://localhost:3000/api/dashboards/uid/canary-traffic-split-day21" | jq -r '.dashboard.title'

# 送測試流量（一般使用者 + 帶 x-canary 的 QA 流量）
for i in $(seq 1 20); do curl -s -o /dev/null -H "Host: demo-app.example.com" "http://<APISIX 對外 IP>/api/hello"; done
for i in $(seq 1 20); do curl -s -o /dev/null -H "Host: demo-app.example.com" -H "x-canary: true" "http://<APISIX 對外 IP>/api/hello"; done

# 對照 PromQL，確認流量分配面板真的查得到兩個版本
kubectl port-forward svc/kube-prometheus-stack-prometheus -n observability 9090:9090
curl -s -G "http://localhost:9090/api/v1/query" \
  --data-urlencode 'query=sum(rate(http_server_request_duration_seconds_count{service_name="demo-app"}[2m])) by (service_version)' \
  | jq -r '.data.result[] | "\(.metric.service_version): \(.value[1])"'
```

## 參考資料

- [Grafana — The RED Method: How to Instrument Your Services](https://grafana.com/blog/the-red-method-how-to-instrument-your-services/)：Tom Wilkie 2015 年提出 RED Method 的官方說明，與 USE Method、Google 四大黃金信號的比較。
- [Grafana — Add and manage dashboard variables](https://grafana.com/docs/grafana/latest/dashboards/variables/)：模板變數的官方文件，`$version` 用的就是這個機制。
