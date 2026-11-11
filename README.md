# Enterprise Cluster Management Platform (ECMP)

一個專為公司內部及外部基礎設施打造的雲原生叢集管理平台（Cluster Management Platform），基於 **[Grit Framework](https://gritframework.dev/)** 構建，統一管理底層 **Talos Linux** 節點（OS 層）與上層 **Kubernetes** 工作負載（K8s 層）。

---

## 核心特色 (Key Features)

* **雙層架構統一管理 (Talos OS + Kubernetes)**：
  * **Talos Linux 控制**：透過原生 Go gRPC API 直接管理 `cp-01` ~ `wk-03` 節點，支援無 SSH 狀態監控、硬碟 S.M.A.R.T 與 M.2 槽位檢查、宣告式 `MachineConfig` 補丁派發、安全重啟及 `etcd` 自動快照。
  * **Kubernetes 資源調度**：整合 `client-go` 統一檢視 `int-cluster`（內部業務叢集）與 `ext-cluster`（外部客戶專案叢集）的節點與工作負載狀態。
* **Oroblanco 標準化命名與一鍵開通精靈 (Self-Service Provisioning)**：
  * 內建標準模板，一鍵自動建立符合公司規範的 Namespace（如 `web-serving-<app>`、`data-postgres-<app>`、`client-<customer>-<project>`）。
  * 自動配發預設拒絕（Default-Deny）`NetworkPolicy`、`ResourceQuota`、Internal DNS 記錄（`*.oroblanco.internal`）及 `cert-manager` TLS 憑證。
* **開箱即用的企業級全端架構 (Powered by Grit Framework)**：
  * 內建 JWT / OAuth2 / 2FA 驗證（可無縫串接 `auth.oroblanco.internal`）、細粒度角色權限控管（RBAC）、Redis 背景任務排程（Asynq）及視覺化資料庫瀏覽器（GORM Studio）。
  * 支援透過 `grit generate resource` 快速擴充新的基礎設施資源模型與管理介面。

---

## 技術架構 (Tech Stack)

| 層級 | 技術選型 | 用途說明 |
| :--- | :--- | :--- |
| **Meta-Framework** | **Grit Framework** (`single` 或 `triple` 模式) | 全端腳手架、程式碼生成與前後端型別同步 |
| **後端引擎 (Backend)** | **Go (Gin + GORM)** | 高併發 REST API、RBAC 權限驗證與業務邏輯 |
| **底層整合 SDK** | `siderolabs/talos/pkg/machinery/client` + `k8s.io/client-go` | 直接與 Talos gRPC API 及 K8s API Server 通訊 |
| **前端與管理面板** | **React + TypeScript + Tailwind CSS + shadcn/ui** | 現代化叢集監控儀表板與自助開通表單 |
| **資料庫與快取** | **PostgreSQL** (`pg-shared.db.prod.oroblanco.internal`) + **Redis** | 儲存叢集資產清冊、操作審計日誌與非同步任務佇列 |
| **可觀測性與安全** | **Grit Pulse + Sentinel** | 內建 API 效能監控、速率限制與安全防護 |

---

## 專案目錄結構 (Project Structure)

```text
oroblanco-cluster-platform/
├── apps/
│   ├── api/                        # Go 後端服務 (Gin + GORM + Talos/K8s Clients)
│   │   ├── internal/
│   │   │   ├── handlers/           # API 路由處理 (Clusters, Nodes, Namespaces, TalosOps)
│   │   │   ├── services/           # 核心業務邏輯 (K8s 資源生成、Talos gRPC 呼叫)
│   │   │   ├── models/             # GORM 資料模型 (Cluster, NodeAsset, AuditLog)
│   │   │   └── drivers/            # Talos gRPC 連線池與 K8s Client 封裝
│   │   └── cmd/server/main.go      # 後端進入點
│   ├── web/                        # 員工自助服務入口 (申請新 App / 專案環境)
│   └── admin/                      # 平台管理員控制台 (Talos 節點維護、Longhorn 監控)
├── deploy/
│   └── k8s/                        # 部署至 infra-cluster-mgmt Namespace 嘅 K8s Manifests
├── docker-compose.yml              # 本地開發環境 (Postgres, Redis, MinIO, Mailhog)
└── turbo.json                      # Monorepo 建置管線設定
```

---

## 快速開始 (Quick Start)

### 1. 前置需求 (Prerequisites)
* **Go** 1.23+
* **Node.js** 20+ & **pnpm**
* **Grit CLI** (`curl -fsSL https://gritframework.dev/install.sh | sh`)
* **Docker**（用於本地啟動開發資料庫）
* 目標叢集嘅 `kubeconfig` 與 `talosconfig` 憑證檔案

### 2. 初始化專案與環境變數
```bash
# 複製環境變數範本並自動生成安全密鑰
grit env

# 啟動本地基礎依賴 (PostgreSQL, Redis, MinIO)
docker compose up -d

# 安裝前端與 Monorepo 依賴套件
pnpm install
```

### 3. 設定叢集連線憑證
編輯 `.env` 檔案，填入 Oroblanco 叢集與內部服務設定：
```env
APP_NAME="Oroblanco Cluster Management Platform"
DATABASE_URL="postgres://ocmp_user:password@localhost:5432/ocmp_dev?sslmode=disable"
REDIS_URL="redis://localhost:6379"

# OIDC SSO 整合 (auth.oroblanco.internal)
OIDC_ISSUER_URL="https://oroblanco.internal"
OIDC_CLIENT_ID="ocmp-client"
OIDC_CLIENT_SECRET="your-client-secret"

# 預設 Talos 與 K8s 設定路徑
INTERNAL_KUBECONFIG_PATH="./secrets/int-cluster.kubeconfig"
INTERNAL_TALOSCONFIG_PATH="./secrets/int-cluster.talosconfig"
```

### 4. 啟動開發伺服器
使用單一指令同時啟動 Go API、前端應用與管理後台：
```bash
grit start
```
啟動後可存取以下服務：
* **Admin 管理後台**：`http://localhost:3001`
* **自助服務入口 (Web)**：`http://localhost:3000`
* **Go API 與互動文件 (Scalar)**：`http://localhost:8080/docs`
* **GORM Studio 資料庫檢視器**：`http://localhost:8080/studio`

---

## 使用 Grit CLI 擴充新功能

當你需要新增管理模組（例如「客戶專案配額紀錄」或「DNS 記錄管理」）時，可直接利用 Grit 程式碼生成器自動建立後端 Model、Migration、API 與前端管理頁面：

```bash
# 範例：生成客戶專案部署紀錄資源
grit generate resource ClientDeployment \
  --fields "customer:string,project:string,cluster:string,namespace:string,domain:string,status:string"
```

---

## 生產環境部署 (Deployment on Talos K8s)

本平台設計部署於內部叢集（`int-cluster`）的專屬基礎設施命名空間中：

* **Namespace**：`infra-cluster-mgmt`
* **內部存取網址**：`https://oroblanco.internal`
* **資料庫連線**：`pg-shared.db.prod.oroblanco.internal` (`ocmp_prod` database)
* **安全權限要求**：
  * 僅綁定最小權限（Least-Privilege）的 Kubernetes `ServiceAccount` 與 Talos `os:reader` / `os:admin` 憑證。
  * 所有修改節點（如 `talosctl reboot`、`upgrade` 或建立 `Namespace`）的操作均會強制寫入不可竄改的審計日誌（Audit Log）並推送至 SIEM 系統。

---

## 開發路線圖 (Roadmap)

- [x] **Phase 1 — 基礎腳手架與多叢集唯讀監控**：整合 `int-cluster` 與 `ext-cluster` 節點清單、Talos 版本與硬體磁碟健康狀態檢查。
- [ ] **Phase 2 — Oroblanco 命名空間與專案自助開通精靈**：一鍵生成 `web-serving-*`、`data-*`、`NetworkPolicy` 與內部 DNS 紀錄。
- [ ] **Phase 3 — Talos 維運自動化**：網頁一鍵執行節點排空（Drain）、Talos OS 滾動升級及排程 `etcd` 快照備份至 NAS。
- [ ] **Phase 4 — Longhorn 儲存整合**：直接於平台檢視 Worker 節點資料碟（NVMe）副本健康度與容量告警。
