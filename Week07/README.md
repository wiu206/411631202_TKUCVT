# W07｜Docker Compose 與資料持久化

## 拓撲圖

```mermaid
flowchart TD
    %% 定義網路區塊
    subgraph NET["Docker Default Network (Bridge)"]
        APP["容器: app\n(Web Application)"]
        DB["容器: db\n(Database)"]
    end

    %% 定義外部資料卷
    subgraph VOLS["Docker Volumes"]
        VOL["Named Volume: db-data"]
    end

    %% 連線與掛載關係
    APP <-->|依賴與內部通訊\nService Name DNS| DB
    DB ===>|掛載於\n/var/lib/postgresql/data| VOL

    %% 樣式美化
    style NET fill:#f9f9f9,stroke:#333,stroke-width:2px,stroke-dasharray: 5 5
    style VOLS fill:#fff,stroke:#333,stroke-width:1px
    style APP fill:#dbeafe,stroke:#1e40af,stroke-width:2px
    style DB fill:#d1fae5,stroke:#065f46,stroke-width:2px
    style VOL fill:#fef3c7,stroke:#92400e,stroke-width:2px
   ``` 
## 從 docker run 到 compose.yaml
## 從 docker run 到 compose.yaml

我最有感的一個改善是：**「再也不用手動去敲 `docker network create` 建立橋樑，而且多容器之間的網路連線變得無腦且直覺。」**

以前用傳統的命令式 `docker run` 時，為了讓 App 容器能連上 Database 容器，我得先手動建一個自訂網路，然後在起每一台容器時小心翼翼地敲上 `--network lab-net`。只要中間漏掉、順序錯了，或者是密碼不小心打錯字，整個連線就會直接斷掉報錯，排錯時還要自己去查 IP。

換到 `compose.yaml` 的宣告式世界後，Docker Compose 直接幫我自動管好網路。它會自動建出一個 default network，最厲害的是**服務名稱（Service Name）直接就能當作內部通訊的 DNS 網域**！我的 `app` 想要連到資料庫，在環境變數直接填 `DB_HOST=db` 就能完美通訊。這種把一整坨複雜的網路指令收納進一份結構清爽的 YAML 檔、直接實現「一鍵啟動（`docker compose up -d`）」的體驗，讓部署多個互相依賴的微服務變得超級優雅且不容易出錯。
## 三種掛載對照

| 掛載類型 | 路徑 (host) | 容器砍重起資料還在嗎 | 重啟容器資料狀態 | 適合情境 |
| :--- | :--- | :--- | :--- | :--- |
| **named volume** | `/var/lib/docker/volumes/<name>/_data` <br>(由 Docker 統一完全管理) | **還在** <br> | **資料完全保留**，新容器接上去能直接無縫讀取舊資料。 | **生產環境 (Prod)**<br>適合資料庫 (如 PostgreSQL、MySQL) 或需要永久保存、備份的應用程式資料。 |
| **bind mount** | **由使用者自行指定** 的任意絕對路徑 <br>(例如 `/home/user/app` 或 `./src`) | **還在** <br> | **資料完全保留**，且 Host 端若有修改，容器內會**即時同步**變更。 | **開發環境 (Dev)**<br>適合將本地原始碼掛載進容器，達成「改程式碼免 rebuild 立即生效」的熱重載。 |
| **tmpfs** | **不存在硬碟上**<br>(直接掛載在 Host 的 **暫存記憶體 (RAM)** 中) | **不在了** <br> | **資料全滅（消失）**，重啟後是一個全新的空目錄。 | **敏感資料或高速暫存**<br>適合存放不能落盤的機密金鑰、憑證，或是追求極致讀寫速度的暫存快取 (Cache)。 |

## healthcheck 前後對照

| 寫法 | curl /healthz t=1s | t=3s | t=5s | t=10s |
| :--- | :--- | :--- | :--- | :--- |
| **只 depends_on** | **500 Internal Error**<br>(或 Connection Refused) | **500 Internal Error**<br>(app 已崩潰或重試中) | **200 OK**<br>(若 app 有自動重連機制此時才正常) | **200 OK** |
| **service_healthy** | **連線無回應 / 阻斷**<br>(app 根本還沒被啟動) | **連線無回應 / 阻斷**<br>(app 依然在排隊等待) | **200 OK**<br>(db 初始化完成變健康，app 隨即順利啟動) | **200 OK** |

### 觀察（用自己的話寫）：
1. **只寫 `depends_on` 的盲點**：
   傳統的 `depends_on` 只是個「弱依賴」。當 $t=1s$ 時，Docker 看到 `db` 容器的 PID 跑起來了，就天真地以為任務完成，立刻把 `app` 也拉起來。但這時候 `db` 還在內部載入設定檔與建立資料庫（這需要幾秒鐘），導致 `app` 一啟動去撞 `db` 就直接撞牆噴錯（500 錯誤或直接掛掉）。

2. **搭配 `condition: service_healthy` 的威力**：
   換成 `service_healthy` 之後，變成「強依賴」。在 $t=1s$ 到 $t=3s$ 期間，因為 `db` 還沒通過健康檢查，Docker Compose 會硬生生按住 `app` 不讓它啟動（所以這時 `curl` 應用程式會完全連不上）。直到大約 $t=5s$、`db` 順利完成初始化並回報健康之後，`app` 才被安全地放行啟動。這完美解決了多容器微服務「起太快而連不上、互相踩踏」的經典時序衝突問題（Race Condition）。

## 排錯紀錄

- **症狀：** 執行 `docker compose up` 啟動服務時，`db` 容器正常運行，但 `app` 容器在啟動瞬間直接噴出 `Connection refused` 錯誤並閃退崩潰，即使在 `compose.yaml` 中加了 `depends_on: - db` 也完全沒有改善。

- **診斷：** 這是經典的容器啟動時序 Race Condition 問題。
  雖然 `compose.yaml` 中寫了 `depends_on`功能，但 Docker 預設只檢查 `db` 容器的進程（Process）是否跑起來，一跑起來就判定「db 已就緒」並放行 `app`。然而此時資料庫在內部其實還在載入設定檔、進行初始化（通常需要數秒），根本還無法接受外部連線，這才導致 `app` 一頭撞上去直接連線失敗而掛掉。

- **修正：** 在 `compose.yaml` 中為 `db` 服務加入真正的健康檢查（`healthcheck`），並將 `app` 的依賴條件升級為強依賴。
  ```yaml
  services:
    db:
      image: postgres:16
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U postgres"]
        interval: 3s
        timeout: 3s
        retries: 3
        
    app:
      image: myapp:v1
      depends_on:
        db:
          condition: service_healthy
- 驗證：重新執行 docker compose up -d。
使用 docker compose ps 觀察狀態，發現 app 容器會先維持在排隊狀態，直到 db 容器的狀態從 (starting) 轉為 (healthy) 之後，app 才被正式拉起來。接著查看 docker compose logs，確認 App 順利連上資料庫並成功提供服務，且經由聯動測試，即使執行 docker rm 強制重啟容器，掛載在 Named Volume 上的資料庫檔案依然完好如初，排錯與持久化驗證成功！

## 設計決策

本週在規劃資料庫（db）的儲存架構時，核心的技術選擇與取捨如下：

### 1. 為什麼 db 選擇 named volume，而不是 bind mount？
在生產環境（Prod）中，將資料庫資料夾掛載出來時，選擇 `named volume` 主要基於以下兩大原因：
* **跨平台的可移植性（Portability）**：`bind mount` 極度依賴 Host（宿主機）的特定絕對路徑（例如 `/home/user/db-data`）。如果換到不同的伺服器、Windows 或 Mac 環境，這個路徑很可能不存在或不正確，導致 Docker Compose 檔案失效。而 `named volume` 的底層路徑完全交由 Docker 引擎自動管理，換到任何環境都能直接「一鍵啟動」，確保了配置檔案的通用性。
* **安全性與權限管理（Permissions）**：資料庫服務（如 PostgreSQL）內部運作時通常使用特定的非 root 使用者（如 `postgres` uid:999）。如果使用 `bind mount`，Host 本地目錄的讀寫權限往往與容器內部不一致，經常導致資料庫啟動時噴出 `Permission denied` 錯誤。`named volume` 在建立時會自動由 Docker 引擎初始化並校正權限，大幅減少權限踩坑的機會。

### 2. 為什麼不能在生產環境用 tmpfs 存資料庫？
* **資料生命週期的致命缺陷**：`tmpfs` 的本質是「直接將資料掛載在 Host 的**暫存記憶體（RAM）**中」，它完全不落盤（不寫入硬碟）。
* **生產環境的災難情境**：在生產環境中，資料庫最重要的指標就是資料的安全性與持久化。一旦伺服器意外斷電、Linux 系統重啟，或者僅僅是有人執行了 `docker compose restart/down`，存在 `tmpfs` 裡的測試檔案或使用者交易資料就會**瞬間灰飛煙滅、100% 徹底消失**，這在生產環境是絕對無法容忍的災難。因此，`tmpfs` 只適合拿來放暫存的快取（Cache）或不能落盤的敏感金鑰，絕不能用來存放需要永久保存的資料庫核心檔案。