# Day 28：Dashboard as Runbook

## 今天的工作

凌晨三點，PagerDuty 把你叫醒：release-decision-service 已經自動退版了，流量已經切回 Stable。但腦子還沒清醒的你，需要立刻回答三個問題：**現在發生了什麼？為什麼觸發？我現在該做什麼？**

傳統做法是打開 Grafana 看指標，再另開一個分頁翻 Confluence 找 SOP——這個切換在凌晨三點多花的 5 分鐘，很容易出錯。今天要做的事，是讓這三個問題的答案都出現在同一個 Grafana Dashboard 上：

1. 介紹 Dashboard as Runbook 的概念，以及三層資訊架構怎麼設計
2. 用 GitOps 方式部署這個 Dashboard
3. 真實注入故障，走一次完整的演練

## 1. 什麼是 Dashboard as Runbook

Runbook（操作手冊）是 SRE 文化中記錄「遇到這個問題該怎麼處理」的文件，傳統上是 Wiki 頁面或 Confluence 文章，跟監控系統完全分離。

**Dashboard as Runbook** 是把 Runbook 的 SOP 文字，直接嵌進 Grafana Dashboard 的 Text Panel 裡，跟指標圖表並排呈現。工程師打開同一個頁面，同時看到「現在的數字」和「這個數字代表什麼、該怎麼反應」，不需要切分頁、不需要在凌晨三點翻文件。

這麼做還有一個好處：Grafana Dashboard 本質上是一份 JSON，存進 Git 之後，Runbook 的每次修改也跟著有版本控制、可以 Code Review——跟 Wiki 頁面比起來，更新紀錄不會斷在某次「隨手改一下」。

## 2. 三層資訊架構：30 秒讀懂現場

一個有效的 Runbook Dashboard，按「閱讀速度」分三層排列：

```text
第一層（5 秒）：現在的狀態是什麼？
  → Stat Panel：Canary 實際流量 %、Burn Rate 倍率、Stable 錯誤率

第二層（10 秒）：什麼時候開始的？嚴重嗎？
  → Timeseries：Burn Rate 趨勢、流量分佈、Canary vs Stable 錯誤率對比

第三層（15 秒）：我現在該做什麼？
  → Text Panel：退版後 SOP、手動晉升 SOP、緊急升級對象
```

這個順序對應人在緊急情況下的認知優先序：先確認「現在有沒有事」，再理解「什麼時候開始的」，最後才是「我的下一步」。版面設計跟人類的思考流程對齊，才是好的 Runbook Dashboard。

第一層的 Burn Rate 倍率 Panel，門檻沿用 Day24／Day26 定的兩個數字：< 0.5x 綠色（可晉升）、0.5x ～ 3x 黃色（觀察中）、> 3x 紅色（退版觸發線），第二層的趨勢圖也用同一組數字畫兩條參考線。

## 3. Panel 怎麼查

九個 Panel 全部查應用層的 `http_server_request_duration_seconds_count`，依 `service_version` 分組。

以「Canary 實際流量 %」這個 Panel 為例：

```promql
sum(rate(http_server_request_duration_seconds_count{service_name="demo-app",service_version="v2-canary",http_route!="/api/health"}[5m]))
/
sum(rate(http_server_request_duration_seconds_count{service_name="demo-app",http_route!="/api/health"}[5m]))
* 100
```

`http_route!="/api/health"` 排除 K8s liveness/readiness probe 固定打的路徑，避免探測流量稀釋真實流量的佔比。

其餘 Panel 用同一個指標，換不同的 `service_version`／時間窗口組合：Burn Rate 趨勢圖疊加兩條參考線（紅線 3x、綠線 0.5x），流量分佈跟錯誤率對比各自把 Canary 跟 Stable 的曲線並排。第三層的三個 Text Panel 放的是純 Markdown：退版後五步 SOP（含確認止血、查 log、確認 commit、修復重部署、重啟 Canary 流程）、手動晉升的兩道確認條件、還有一張「誰也修不好時該找誰」的升級表。

## 4. GitOps 部署：ConfigMap + Grafana Sidecar

Dashboard JSON 存進 Config Repo 而不是用 UI 手動建立，原因跟管理 OPA ConfigMap 一樣：每次修改有版本控制，出問題可以 `git revert`。

實作方式是 Grafana 內建的 sidecar container（`kiwigrid/k8s-sidecar`），它會 watch 叢集內帶有 `grafana_dashboard: "1"` label 的 ConfigMap，自動把 JSON 匯入成 Dashboard，不需要碰 Grafana UI：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-canary-runbook
  namespace: observability
  labels:
    grafana_dashboard: "1"   # Sidecar 讀取此 label，自動匯入
data:
  canary-runbook.json: |
    { ... Grafana Dashboard JSON ... }
```

ArgoCD 偵測到 Config Repo 的新 commit → 同步 ConfigMap → Grafana Sidecar 自動匯入 → Dashboard 立即可用，跟 OPA ConfigMap 的管理模式完全一致。

跟 Day26／27 不一樣的是，這次**不需要**新建 ArgoCD Application——`dashboards/` 這個路徑本來就被既有的 `grafana-dashboards-app` Application 涵蓋，今天唯一要做的 GitOps 動作，是把這份 ConfigMap 從暫放的 `dashboards-staging/` 搬進正式的 `dashboards/`。
