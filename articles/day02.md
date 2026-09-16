# Day02：建立 CI/CD pipeline

## 今天的工作

1. 我們要在 GitHub 建立一個 2 個 git repo，一個用來放 demo-app，另一個用來放 ArgoCD 需要的設定。
2. 建立 CI/CD pipeline。

## 由 2 個 git repo 組成的 CI/CD

### 2 個 git repo 的責任

- demo-app: 用來放 demo-app，由 Java 編寫。

- config: 用來放 ArgoCD 佈署所需的設定檔。ArgoCD 會在 K8s 裡佈署我們需要的架構，包含 APISIX、demo-app 和 middle-app。

| | Demo-App Repo | Config Repo |
| --- | --- | --- |
| **存放內容** | 業務源碼、Dockerfile、CI 工作流程 | K8s YAML、Helm values、APISIX 路由規則 |
| **變更觸發者** | 開發者的每一次 Commit | CI 完成後的跨倉庫更新（僅更新 image tag） |
| **監聽者** | CI Pipeline | ArgoCD |
| **變更頻率** | 高 | 低 |
| **寫入權限** | 所有開發者 | CI Bot + SRE / Platform Team |

### CI/CD pipeline 合作關係

- CI：每次對 "demo-app" repo 執行一次 git commit，都會產生一個新的 "app 版本號"，然後這個版本號會被推送到另一個 git repo "config"

- CD：
  - 定時輪詢：K8s 裡的 ArgoCD 固定時距會來讀取 config 裡的設定，根據當時的設定調整 K8s 裡的資源。本專案設定的時距是 60 秒。
  - webhook 觸發：每次 "config" repo 特定目錄裡的檔案發生修改，都會用 webhook 的方式觸發 ArgoCD，讓 ArgoCD 來比對改變後的檔案，調整 K8s 裡的資源，例如：demo-app、APISIX、OPA service 和 ApisixRoute.rego。
  > 考慮到 webhook 的複雜度，我們在專案裡不採用。只使用定時輪詢。

## 拆解 CI

### 4 個步驟

```text
開發者推送業務代碼
        ↓
[CI Step 1] Maven 編譯 + 單元測試
        ↓
[CI Step 2] Docker Build
        ↓
[CI Step 3] Docker Push 到 GCP Artifact Registry
        ↓
[CI Step 4] 跨倉庫更新 config Repo 的 image tag（跨倉庫寫入）
        ↓
config Repo 發生變更，ArgoCD 定時輪詢偵測到變更，自動同步至 K8s
```

### 3 個 Job

| Job | 執行時機 | 職責 |
| --- | --- | --- |
| `build-test` | 所有 Push 與 PR | Maven 編譯 + 單元測試，PR（Pull Request）的品質門控 |
| `docker-push` | 僅 Push 到 `main` / `canary` | Docker Build & Push 到 Artifact Registry |
| `update-config-repo` | 接續 `docker-push` 完成後 | 跨 Repo 更新 Config Repo image tag，觸發 ArgoCD 同步 |

### 需要申請的 GCP 權限

我們要在 GCP 建立一個 Service Account，讓 App repo 可以把 docker image 推到 GCP Artifact Registry。

如果不想用 Service Account，另一種做法是把一組 GCP 的金鑰檔案(JSON Key)存進 GitHub Secrets，讓 CI 直接用這組金鑰推 image。問題是這組金鑰是長期有效的，一旦外洩，攻擊者可以不斷把 docker image 塞進你的 GCP Artifact Registry，一個晚上的費用就可以讓你懷疑人生，從此不碰公有雲。

所以我們採用「事先建立一個專屬 Service Account + 只給它剛好夠用的權限 + 讓 GitHub 每次臨時借用」的做法，將
權限最小化，讓這個 Service Account 只被授予「推 image 到這一個 Artifact Registry 倉庫」的權限，沒有其他多餘能力
。整個過程沒有任何固定金鑰檔案存在任何地方，並且一個小時就過期，減少懷疑人生的機會。

#### 建立流程  

1. 建立一個專屬的 Service Account "ci"：  
把「推 image 到這個 Artifact Registry 倉庫」的權限(roles/artifactregistry.writer)，只綁定在 "demo-app" 這一個倉庫上，不是整個 GCP 專案。  
2. 建立一個 Workload Identity Pool(github-pool)：  
一個專門用來裝「外部身份」的容器。在這個 Pool 裡建立一個 Provider(github-provider)，設定成只信任 GitHub 官方簽發的證件，並且要求證件上寫的 repo 名字要對得上這個專案的 repo。這些步驟可以用 terragrunt 來完成，git repo 的連結會放在 Day30。

#### 驗證流程

GitHub Actions 執行時（CI 階段），會跟 GitHub 要一張證件(OIDC token)，上面寫著這次是哪個 repo、哪個 workflow 跑的。
這張證件被拿去問 GCP「根據這張證件，授權給我權限」。  

1. GCP 先檢查：這張證件真的是 GitHub 簽的嗎？上面寫的 repo 名字對不對？（第 1 點裡 Provider 設定的那一關）
2. 通過之後，GCP 再檢查：這個身份有沒有資格借用 ci 這個 Service Account？（第 1 點裡最後一步設定的那一關）
3. 兩關都過，GCP 發一張效期最長 1 小時的臨時通行證，代表暫時借到了 ci 這個 Service Account 的身份
CI 拿這張臨時通行證去 docker push，這次執行結束後，通行證自動失效。

### 需要建立的 GitHub 權限

我們需要建立一個 GitHub Apps，安裝在 config repo 上，權限是「讀&寫」，讓 demo-app repo 可以調整 config repo 上的 image tag。

## 拆解 CD

### 佈署 ArgoCD Helm Chart

本專案使用官方維護的 ArgoCD Helm Chart `v9.5.13`。`helm install argocd ...` 這條指令，默認會佈署多組 Deployment，我們會用到以下的子服務：

| 子服務 | 職責 | HA 方式 | 本專案副本數 |
| --- | --- | --- | --- |
| `argocd-server` | Web UI／CLI／API 後端，RBAC 授權判斷入口 | Stateless，直接疊 replica | ×1 |
| `repo-server` | Clone Git repo，跑 Helm/Kustomize/純 YAML 引擎，把宣告式配置渲染成實際 K8s manifest | Stateless，直接疊 replica | ×1 |
| `application-controller` | 核心 Reconciliation Loop：比對 `repo-server` 渲染出的期望態 vs 叢集現況，決定要不要 Apply | Stateless，多複本會啟用 sharding（靠 redis-ha 協調） | ×1 |
| `redis-ha` | 快取 + `application-controller` 的分片狀態儲存 | Redis Sentinel 主從選舉，機制與其他三者不同 | ×3 |

> **[副本數歸零]**：ArgoCD 標準的 Helm chart 預設還會裝第 4 個以上的子服務。由於部份子服務沒有提供「不裝這個元件」的開關，本專案把用不到的子服務 `replicas` 設成 `0`，Deployment 物件還在，但不會有 Pod 真的在跑，效果等於關閉。

`argocd-server`／`repo-server`／`application-controller` 這三個雖然都支援水平擴展，但本專案是 PoC 規模，沒有高並發或容錯演練的場景——`application-controller` 開 2 個複本以上還會啟用 sharding（每個複本各自負責一部分 Application），反而讓「這個 Application 的同步狀況該看哪個 Pod 的 log」變得更難追蹤，所以三個都只開 1 個複本，換取除錯時的簡單直覺。

以上子服務的副本數，對應 `values.yaml` 的實際部署配置：

```yaml
controller:
  replicas: 1
repoServer:
  replicas: 1
server:
  replicas: 1
redis-ha:
  replicas: 3
```

### 需要建立的 GitHub 權限

在 GitHub 建立 GitHub App "enterprise-gitops-argocd"，安裝到 config repo 上，權限只給 "唯讀"，因為 ArgoCD 只需要讀。

### 設定 GCP 和 ArgoCD 的權限

- GCP 這邊：建立 GitHub App "enterprise-gitops-argocd" 之後，會拿到三樣東西—— **App ID、Installation ID、私鑰**。這三樣東西存進 **GCP Secret Manager**，變成三個 Secret。這裡的處理方式跟 CI 那邊的機制不一樣。

- ArgoCD 這邊：從 GCP Secret Manager 把這三個值讀出來，跑以下 2 條指令，包成一個 K8s Secret(config-repo-creds)，放進 `argocd` 這個 namespace，並貼上一個特定的 label `argocd.argoproj.io/secret-type=repo-creds`：

```bash
# 這是一次性的叢集 bootstrap 操作，不是每次程式碼變更都要做的事，所以用指令完成
kubectl create secret generic config-repo-creds \
  --namespace argocd \
  --from-literal=type=git \
  --from-literal=url="<config repo 網址>" \
  --from-literal=githubAppID="<App ID>" \
  --from-literal=githubAppInstallationID="<Installation ID>" \
  --from-literal=githubAppPrivateKey="<私鑰>"

kubectl label secret config-repo-creds \
  -n argocd \
  argocd.argoproj.io/secret-type=repo-creds
```

`argocd.argoproj.io/secret-type=repo-creds` 這個 label，告訴 ArgoCD「這是拿來連 Git repo 用的憑證」。

實際運作時，repo-server 要去 clone config 這個 repo 之前，會去讀這個 K8s Secret 裡的私鑰，拿去跟 GitHub 換一張效期 1 小時的臨時 token，再用這張 token 去 clone repo。

## 參考資料

ArgoCD 官方 Webhook 設定教學（本專案評估後未採用，見上方「拆解 CD」的取捨說明，這裡列出來供對照）：

- Webhook Configuration
  <https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/>

GitHub Apps 官方教學：

- Registering a GitHub App
  <https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app>
