# Day 14：一行代碼切斷金絲雀災難——GitOps 宣告式回滾

## 今天的工作

1. 認識 Kill Switch：一個全域開關，出事故時不用碰任何 Deployment/kubectl，就能瞬間切斷金絲雀流量
2. 在 Day08-11 已經寫好的 Rego 之外，新增一個 `canary.json`，把這個開關值跟判斷邏輯拆開放
3. 演練一次真正的故障：切開關 → 驗證 → 切回來
4. 收尾 Stage 2（Day08-14）

今天要解決一個問題：金絲雀壞了，怎麼在最短時間內把流量全部拉回 Stable。

## 為什麼不能手動 kubectl 救火

金絲雀版本壞了（假設開始回 500），直覺的救火方式可能是「縮容」Pod：

```bash
kubectl scale deploy demo-app-canary --replicas=0
```

或者手動改 APISIX 路由，把全部流量導入 Stable Pod：

```bash
kubectl edit apisixroute
```

這個做法在這個叢集裡有個具體的問題：  
這兩個指令動到的資源（`demo-app-canary` 的 Deployment、`apisixroute`）都是 `demo-app-app.yaml` 這個 ArgoCD Application 在管，而它設了 `selfHeal: true`。也就是說，手動改動一下去，ArgoCD 下一次 reconcile（設定是 60 秒一次）就會偵測到「叢集實際狀態」跟「Git 宣告的狀態」對不上，直接把手動改動覆蓋回去，於是一分鐘後我們又會回到原來的故障情況。

我們從 Day01 開始就把專案打造成 **GitOps 架構**，透過 git 管理全部的佈署，所以正確並且有用的救火方式：改 Git，讓 ArgoCD 把這個改動當成新的「宣告狀態」同步下去。

## Kill Switch：加一個總開關，跟判斷邏輯分開放

回顧 Day10、Day11 寫好的兩條 `is_canary` 規則：

```rego
is_canary {
    input.request.headers["x-canary"] == "true"
}

is_canary {
    contains(input.request.headers["cookie"], "canary=true")
}
```

今天要加一個全域開關，兩條規則都要先過這一關。這個開關會被救火時頻繁改動（正常時 `true`，出事故時改 `false`），跟幾乎不變的判斷邏輯放在同一個檔案裡不太合適——所以從今天開始，**把「會變動的設定值」跟「不常變的判斷邏輯」拆成兩個檔案**：`canary-routing.rego` 只留邏輯，新增一個 `canary.json` 專門放這種開關值。

`canary.json`：

```json
{"canary": {"enabled": true}}
```

`canary-routing.rego` 改成讀這個外部值，不再自己宣告：

```rego
is_canary {
    data.canary.enabled == true
    input.request.headers["x-canary"] == "true"
}

is_canary {
    data.canary.enabled == true
    contains(input.request.headers["cookie"], "canary=true")
}
```

### 這兩個檔案怎麼「合作」

OPA 內部只有一棵叫 `data` 的文件樹，不管是規則算出來的值、還是外部檔案給的值，最後都會掛進這棵樹裡——OPA 不會特別區分「這是規則」還是「這是資料」兩種類型，只在乎掛在哪個路徑。

- `canary-routing.rego` 開頭的 `package canary`，讓這份檔案裡所有規則掛進 `data.canary` 這個路徑。
- `canary.json` 的內容最外層包一個 `"canary"` key，讓這包資料**也**掛進同一個 `data.canary` 路徑。
- 兩邊指向同一個路徑，OPA 會把內容合併成同一個物件——`data.canary` 底下同時有 JSON 給的 `enabled`，也有 Rego 算出來的 `is_canary`、`headers`。規則裡寫 `data.canary.enabled`，讀到的就是 `canary.json` 給的值。

這也是為什麼 `canary.json` 一定要包這層 `"canary"` key、不能直接攤平寫成 `{"enabled": true}`——攤平的話會掛到 `data.enabled`，跟 Rego 期待的 `data.canary.enabled` 路徑對不上，`is_canary` 會靜默恆為 `false`，不報錯也不 crash，Kill Switch 看起來設好了、實際上完全沒作用。

### 改了值之後，OPA 什麼時候套用

兩個檔案都掛進同一個 ConfigMap，OPA 啟動時帶 `--watch` 參數，個別指名這兩個檔案路徑（不是指向資料夾）。[OPA 官方文件](https://www.openpolicyagent.org/docs/latest/cli/)對 `--watch` 的說明是：「偵測到變化時，**更新後的 policy 和 data 會一起被重新載入**」——不管是改 `canary-routing.rego` 還是 `canary.json`，只要任一個檔案內容變了，OPA 會把兩個檔案的最新內容**一起**重新讀進記憶體、換掉舊的，不是只更新變動的那一個。整個過程發生在同一個 Pod、同一個 process 裡，不用重啟、不用重建 Pod。

### 關閉之後，發生了什麼事

用 Day08 已經講過的「undefined」語意，走一次 `canary.json` 的 `enabled` 改成 `false` 之後的工作邏輯：

1. 不管 request 帶了什麼 Header 或 Cookie，兩條 `is_canary` 規則的第一個條件 `data.canary.enabled == true` 都不成立，於是「規則整條不成立」。
2. `is_canary` 對所有 request 變成 **undefined**（不是 `false`，是這個值根本沒被算出來——Day08 講過的那種「整個欄位消失」）。
3. `headers` 落入 `not is_canary` 那條規則，這條規則裡 **沒有** `x-route-to` 這個 key。
4. `opa` plugin 沒有 `x-route-to` 可以轉發，`traffic-split` 讀到的 `http_x-route-to` 是空的，條件不成立，請求照 `backends` 的預設值落回 `demo-app-stable`。

---

`baggage_val` 和 `headers` 這兩段規則完全不用改動。  
它們是從 `is_canary` 推導出來的，`is_canary` 一旦恆為 undefined，`baggage_val` 自動變成 `routing-context=stable`，跟著連鎖生效。

## 演練：真的切一次

情境：Canary 版本壞了，開始回應 500。

步驟：

1. 把 ArgoCD `opa` Application 的 `source.path` 目前指到的 `manifests/infra/opa/dayNN/opa-deployment.yaml` 裡，`canary.json` 的 `enabled` 改成 `false`（不是改 `canary-routing.rego`），commit、push。
2. 等 ArgoCD 同步 + ConfigMap 傳播到 Pod（這個專案實測過，從 push 到系統真的生效，大約要 3-4 分鐘，push 完不要馬上測）。

   > 這段時間拆開來看有三層：
   > 1. **ArgoCD 偵測到 git 有變更**——本專案固定 60 秒輪詢一次，平均要等半個週期。
   > 2. **kubelet 把更新後的 ConfigMap 同步進 Pod 掛載的 volume**——這是 K8s 自己的週期性機制，跟 ArgoCD 怎麼被觸發完全無關。
   > 3. **OPA 自己用 `--watch` 偵測掛載目錄裡的檔案變了**，才觸發熱重載。
   >
   > 如果把第 1 步換成 [webhook](https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/)，理論上能讓 ArgoCD 幾乎立刻知道 git 有變更，平均省下約 30 秒——但第 2、3 步不會因此變快，這兩步加起來才是大頭。換 webhook 對整體 3-4 分鐘的影響有限，這也呼應了 Day02 沒有採用 webhook 的判斷：複雜度換來的效益，在這個情境下很有限。
3. 用同一組 `curl -H "x-canary: true"` 打過去——這次應該被強制導回 `demo-app-stable`，不是被擋掉、也不是 403，是這個 Header 被整個忽略。

`scripts/verify-day14-kill-switch.sh` 就是照這個邏輯寫的：它是唯讀腳本，不會自己切開關，只讀取 OPA 當下實際載入的 `canary_enabled` 值，依照這個值斷言對應的預期行為——`canary_enabled=false` 時，除了確認 `x-canary: true` 被導回 stable，還會比對 `demo-app-canary` 這個 Pod 的日誌筆數在這次請求前後有沒有增加，證明流量真的一筆都沒有漏過去，不是只看表面回應。

演練做完，記得把 `canary_enabled` 改回 `true`、commit、push——不要讓叢集停在回滾狀態。

## Stage 2 收尾

Day08 到今天，我們用 Policy-driven 的精確路由：Header、Cookie、Baggage 透傳、到今天的 Kill Switch，全部由 OPA 這一層決定，APISIX 只負責照 OPA 的決定執行。

現在整個系統 **沒有任何告警或儀表板**。今天演練「Canary 壞了」這個情境時，是我們自己知道要去切開關。在真實世界裡，第一步永遠是「怎麼發現壞了」。接下來 Stage 3（Day15 開始），我們要進入「可觀測性」的領域，讓重要的訊息與數據透過「圖像」來表達。

## 提醒

- 救火時要改的是 `canary.json` 的 `enabled`，不是 `canary-routing.rego`——改 Rego 檔案不會有任何效果，因為判斷邏輯只負責「讀」這個值，不負責「存」這個值。
- 改 `policies/canary-routing.rego` 不會有任何效果，這份檔案從未部署——真正要改的是 ArgoCD `opa` Application 的 `source.path` 目前指到的 `manifests/infra/opa/dayNN/opa-deployment.yaml`。
- 這個開關是全域的，關閉後 QA 想用 `x-canary: true` 測試 Canary 也會被一起擋下來，沒有例外——這是設計上的取捨：出事故時要的是「全部拉回 Stable」，不是「大部分拉回、留一個後門繼續測」。

## 參考資料

- [OPA 官方文件 — CLI（`--watch`）](https://www.openpolicyagent.org/docs/latest/cli/)：「the updated policy and data is reloaded into OPA」——`--watch` 多檔案時整批一起重載的官方依據；同頁也警告監控個別檔案可能導致更新被忽略。
- [ArgoCD 官方文件 — Automated Sync Policy（Self-Healing）](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/#automated-self-healing)：官方明講手動用 `kubectl edit` 改動受管資源後，只要 `selfHeal` 有開，ArgoCD 偵測到偏移就會自動同步、蓋回 Git 宣告的狀態——這是「手動 kubectl 救火撐不過一分鐘」這個說法的直接依據。
- [ArgoCD 官方文件 — Webhook Configuration](https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/)：官方說明 webhook 的作用只是提前觸發一次跟輪詢相同的 reconcile 流程——這是「webhook 只省得到 ArgoCD 偵測這一步、省不到 ConfigMap 傳播與 OPA reload」這個判斷的依據。
