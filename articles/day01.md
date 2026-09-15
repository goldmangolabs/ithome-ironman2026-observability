# Day01：專案的 4 個階段任務

## 今天的工作

1. 說明專案背景環境與假設
2. 說明專案的 4 個階段與目標
3. 說明代碼片段的閱讀方式

## 專案背景環境

我們要做一個金絲雀的佈署架構，並且隨時監控金絲雀版本的服務健康狀況。當金絲雀的健康狀況合乎期待，系統會自動逐步增加導入金絲雀的 HTTP request 比重；如果金絲雀狀況惡化，系統會立即停止導入 HTTP request，並且自動發出告警通知。一旦系統開始運作，只能透過 git commit 來控制。

我假設這個金絲雀佈署的架構，發生在一間公司內部的 K8s 環境。只有公司內部的人可以訪問到這座 K8s 的服務。為避免模糊焦點，在本專案最低限度討論「資訊安全、備援和災難複原」的議題，這些主題都很棒，很適合單獨拿出來做成專案。要是全塞在一起，反而很難呈現應該有的效果。

## 專案的 4 個階段與目標

整個專案會拆成 4 個階段，每個階段都有一個主要任務。

### 階段一：建立 infra 架構

1. 用 Terragrunt 在 GCP 建立 GKE 叢集等基礎設施
2. 建立 GitOps CI/CD 工作流程
3. 佈署 APISIX 做為流量閘道
4. 佈署同一支服務的 Stable 和 Canary 版本
5. 使用 OTel Agent 零侵入方式標記 request，讓 Canary 跟 Stable 對外都用同一個 OTel `service.name`，只靠 `service.version` 區分兩者，方便後面所有的監控指標可以被分開比較

### 階段二：設定 request 的路由策略

1. 說明 OPA service 的工作邏輯
2. 說明 APISIX 的 plugin——opa plugin 和 OPA service 的合作方式
3. 說明 APISIX 的 plugin——traffic-split plugin
4. 驗證 QA 人員可以透過帶特定的 HTTP Header，精準被導向 Canary 版本
5. 讓內部員工不用手動帶 Header，改用 Cookie 就能自然被導向 Canary 版本（內部驗證測試，Dogfooding）
6. 提出一個問題：跨服務呼叫時，金絲雀身份會遺失
7. 說明 OTel Agent 如何利用 HTTP Header traceparent 和 baggage 來標記 request，解決第 6 點的問題，讓我們可以做到精密監控
8. 引入 middle-app，串出一條跨服務的呼叫鏈，驗證 baggage 真的能跨服務存活
9. 建立一個「立即停止 Canary 被訪問」的功能

### 階段三：建立可觀察性

1. 部署 OTel Collector（Agent + Gateway），把 demo-app 跟 middle-app 的 OTel Agent 資料送過去
2. 部署 Prometheus，驗證資料真的從 Collector Gateway 送到 Prometheus
3. 啟用 APISIX 的 `prometheus` 插件，補上閘道層自己的監控數據
4. 部署 Grafana Tempo，讓 APISIX 也加入 Trace，串出一條涵蓋閘道跟應用層的完整 Trace
5. 部署 Loki 跟 Promtail，讓同一個 `trace_id` 能同時查到閘道日誌跟應用日誌
6. 部署 Grafana，把 Prometheus／Tempo／Loki 接成 Datasource
7. 建立第一張 Dashboard：Canary Traffic Split（金絲雀核心決策看板，RED Method）
8. 建立第二張 Dashboard：APISIX Upstream Health，並用故障注入驗證面板

### 階段四：階段晉升或者停止 Canary 接受的訪問量

1. 在既有的精確比對之上，加入比例分流（依識別值取餘數），並確保同一使用者的黏滯性
2. 引入 SLO／Error Budget／Burn Rate 概念，推導出晉升與退版兩個動態閾值
3. 手動走一次完整的漸進式放量：10% → 50% → 100%，定義每一站的 Bake Time
4. 100% 之後把 Stable 縮編到 1 台，當作緊急回滾的備援，不整個關掉
5. 部署 PrometheusRule，讓 Burn Rate 超標時 Prometheus 主動觸發告警
6. 建立 release-decision-service.py，自動化「主動晉升」與「被動退版」兩條路徑，取代人工 SOP
7. 建立 Dashboard as Runbook，把「現在的狀態」跟「該怎麼處理」放進同一個畫面
8. 真實故障演練，驗證整條「告警 → 自動退版 → Dashboard 反映」的閉環

## 代碼片段的閱讀方式

這是連載式開發：每天文章裡貼的代碼，只呈現當天新增或修改的部分，並且後續天數會持續在同一批檔案上疊加、改寫早期的邏輯（例如比例分流的判斷式，會在流量分流機制確立之後才加進去）。我會在更改的時候註明「正在改變哪一天的代碼」。所以如果你發現某一天講的內容，跟其他天看到的完整代碼對不起來，這是預期中的現象。在 Day30 之後會把最終版本的代碼放到 GitHub 上。
