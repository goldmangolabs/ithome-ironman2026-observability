# Day 05： 同時佈署 demo-app 的 Stable 和 Canary 版本

## 今天的工作

1. 介紹本專案用來展示金絲雀佈署的 app：demo-app
2. 同時佈署 Stable 和 Canary 兩個版本

## demo-app 是什麼？

demo-app 是為這個專案寫的一支極簡 Spring Boot 服務，專門配合金絲雀部署的展示與測試，它只有兩個 API：

- `GET /api/health` — 回傳 `{"status": "UP"}`，給 K8s 的健康檢查用
- `GET /api/hello` — 主要的展示端點，回傳 `{"version": "v1.0.0", "msg": "Hello from demo-app"}`

`/api/hello` 特別的地方是它支援兩個可選參數，讓我們能「按需製造問題」：

- `delay`（毫秒）— 故意讓這次請求延遲指定時間，模擬慢請求
- `error` — 故意丟出例外，模擬失敗

這兩個參數就是後面幾天做金絲雀健康監控、自動回滾演練時要用的「開關」——不用真的等系統壞掉，直接呼叫這個 API 就能製造出想要的故障情境。

## 怎麼佈署 Stable 版本

實際部署的 YAML 放在 `config` repo 的 `manifests/apps/demo-app/demo-app-v1.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app-stable
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-app
      track: stable
  template:
    metadata:
      labels:
        app: demo-app
        track: stable
    spec:
      containers:
        - name: demo-app
          image: <stable image 倉庫位置>/demo-app:<tag-stable>
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: demo-app-stable
spec:
  selector:
    app: demo-app
    track: stable
  ports:
    - port: 80
      targetPort: 8080
```

## 怎麼佈署 Canary 版本

Canary 版的 YAML 幾乎是 Stable 的鏡像，放在同一個資料夾的 `demo-app-v2-canary.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-app
      track: canary
  template:
    metadata:
      labels:
        app: demo-app
        track: canary
    spec:
      containers:
        - name: demo-app
          image: <canary image 倉庫位置>/demo-app:<tag-canary>
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: demo-app-canary
spec:
  selector:
    app: demo-app
    track: canary
  ports:
    - port: 80
      targetPort: 8080
```

## 幾個關鍵點

- `track: stable` / `track: canary` 這個 label，用來區分佈署後的 Pod 是 Stable 還是 Canary 版本。
- Service 的 `port: 80` 跟 Pod 的 `containerPort: 8080`：`port` 是這個 Service 對外曝露的埠，`targetPort` 才是真正轉發進 Pod 容器裡的埠。Day04 那條 `ApisixRoute` 裡 `servicePort: 80`，指的就是這裡的 `port`。
- 兩邊的 `image` tag（<tag-stable>，<tag-canary>）是 git commit 的短 SHA。這兩支 image 分別從 `demo-app` repo 的 CI 建出來。**push 到 `main` 分支**觸發建置、標記給 Stable；**push 到 `canary` 分支**觸發另一次獨立建置、標記給 Canary。CI 依「是哪個分支觸發的」決定要更新這裡哪一份 YAML 的 image tag，兩邊的版本各自跟著自己分支的最新進度走，不需要我們手動改。
