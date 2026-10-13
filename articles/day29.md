# Day 29：一次假想故障，走一遍整個系統

## 今天的工作

1. 用一次假想的 Canary 故障當主軸，把 Day01 到 Day28 建的所有機制，串成一張完整的大圖
2. 釐清圖裡容易搞混的一個細節：同樣是跟 Prometheus 要資料，「告警」跟「自動放量」用的是完全相反方向的機制

## 一筆 Canary 流量，到故障，到自動修復

下面這張圖，是整個故事的全貌：

![Day29 架構圖](架構圖/day29.png)

### ① 正常流量路徑（Day04-14、Day23）

Client 打進 APISIX，APISIX 透過 `opa` plugin 查詢 OPA——這一步圖上標了一個 Ⓐ，後面會用到。OPA 判斷這筆請求該不該進 Canary，靠的是兩層機制 OR 合併的結果：`x-canary` Header／`canary=true` Cookie 的精確比對（Day08-11），或是 `x-user-id` 雜湊取餘數跟 `weight` 比大小（Day23）。判斷完回傳 `x-route-to`，APISIX 的 `traffic-split` 照這個值把請求送進 Stable 或 Canary 的 upstream。

### ② 可觀測性管線（Day15-22）

不管請求落在 Stable 還是 Canary，這兩個版本的 OTel Java Agent 跟 APISIX 自己的三個插件（Day17-19），都會吐出 Metrics／Logs／Traces，送進 OTel Collector，再分流進 Prometheus、Tempo、Loki。這一層不是故障發生才啟動的，是從 Day15 開始，每一筆請求背後都默默在跑的。Grafana（Day20）把三者收斂成一個查詢介面，Day21、22、28 的三張 Dashboard 都建在這個基礎上。

今天的情境假設：這次放量後，Canary 自己的程式碼出問題，它收到的請求裡有 20% 開始回應錯誤——這個錯誤率會被 Prometheus 量到。

### ③ 自動化決策迴路（Day23-28），而且是雙向的

這一塊是整個系列最後拼起來的部分，是兩條方向相反、但共用同一組「改寫 `canary.json` → commit → ArgoCD 同步 → OPA 熱重載」機制的迴路：

- **被動（Day26-27）**：Prometheus 自己評估 PrometheusRule，算出 Burn Rate 持續超過 3x 滿 2 分鐘，**主動推送（push）**一個 FIRING 的告警給 Alertmanager，路由到 `release-decision-service` 的 webhook，把 `canary.json` 改成 `enabled=false, weight=0`。
- **主動（Day24-25、27）**：`release-decision-service` 自己的 Promotion Loop，每 60 秒**自己醒來、主動去查詢（pull）**Prometheus，確認 Bake Time 夠久、Burn Rate 夠低，就把 `weight` 推進下一階（10→50→100）。

這兩條路徑推送跟拉取的方向完全相反——左邊是 Prometheus 主動找上門，右邊是 `release-decision-service` 自己伸手去問——但殊途同歸，最後都走同一條 commit 回 Config Repo、ArgoCD 同步、OPA 熱重載的路。

### 回到 Ⓐ：迴圈怎麼接起來

OPA 熱重載之後，從下一筆請求開始，APISIX 問到的 Ⓐ 就是新答案了——不管是退版後的「全部回 Stable」，還是放量後的「更高比例進 Canary」。這就是為什麼圖上 OPA 只畫一次：它既是①平常被 APISIX 即時查詢的對象，也是③自動化迴路最後更新的對象，同一個 OPA，兩種身份。

## 提醒

- 今天沒有新的 YAML 或 CRD，純粹是把前 28 天已經各自驗證過的機制，串成一張完整的因果鏈。
