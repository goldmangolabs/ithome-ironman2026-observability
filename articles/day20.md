# Day 20：部署 Grafana，接上 Metrics/Traces/Logs 三個 Datasource

## 今天的工作

1. 部署 Grafana
2. 把 Prometheus／Tempo／Loki 接進 Grafana 成為 Datasource

今天我們把 Prometheus、Tempo、Loki 三個工具接進 Grafana 成為 Datasource。

## 部署 Grafana

Helm `kube-prometheus-stack` 是一個「大禮包」Chart，同時打包了 Prometheus、Alertmanager，還有 Grafana。`grafana.enabled: true` 會讓 Grafana 的 Deployment、Service、ConfigMap 等資源，跟著 Prometheus、Alertmanager 一起，用 **同一個 Helm Release** 渲染出來：

```yaml
# manifests/infrastructure/prometheus-stack/values.yaml
grafana:
  enabled: true
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard
      labelValue: "1"
```

## 把 Prometheus／Tempo／Loki 接成 Grafana 的 Datasource

分成 "自動" 和 "手動"。

### Prometheus：自動接入，不用設定

因為 Prometheus 跟 Grafana 是同一個 Helm chart（同一個 Release）裝出來的，chart 自己的 template 就能直接生成一個指向「自己裝的 Prometheus」的 Datasource，不用另外設定。

### Tempo／Loki：手動接入

Tempo 跟 Loki 不屬於 `kube-prometheus-stack` 管理的範圍，需手動接入：

```yaml
# manifests/infrastructure/prometheus-stack/values.yaml
grafana:
  additionalDataSources:
    - name: Tempo
      type: tempo
      access: proxy
      url: http://tempo.<namespace>.svc.cluster.local:3200
    - name: Loki
      type: loki
      access: proxy
      url: http://loki.observability.svc.cluster.local:3100
```

Tempo 和 Loki 的 URL：

- 查看 Tempo 和 Loki 的 K8s Service
- 從 Service 拚出 URL `<Service 名稱>.<namespace>.svc.cluster.local:<port>`

## 驗證：能登入 Grafana

**驗證策略：**

- 確認 `kube-prometheus-stack` 這個 ArgoCD Application Synced/Healthy
- 確認 Grafana 的 Deployment 真的 Ready
- 用以下指令連通本機與 Grafana，實際登入看看

```bash
# 取得 Grafana admin 密碼
kubectl get secret kube-prometheus-stack-grafana -n observability \
  -o jsonpath='{.data.admin-password}' | base64 -d

# port-forward 到 Grafana
kubectl port-forward svc/kube-prometheus-stack-grafana -n observability 3000:80
```

- 打開瀏覽器，輸入 URL：`http://localhost:3000`。登入 Grafana。

## 參考資料

- [Helm — Tags and Condition Fields in Dependencies](https://helm.sh/docs/topics/charts/#tags-and-condition-fields-in-dependencies)：`condition` 欄位讓父 Chart 有條件啟用子 Chart 依賴的官方機制說明，`grafana.enabled` 背後就是這個機制。
