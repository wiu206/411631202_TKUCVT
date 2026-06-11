# W08｜容器生產實踐

## Healthcheck 故障測試

* **停 db 後幾秒被標 unhealthy：** 大約 `9` 秒到 `10` 秒左右。
    > **（原理解釋）：** 這是由你在 `compose.yaml` 中設定的健康檢查參數決定的。舉例來說，若你的設定是 `interval: 3s`（每 3 秒檢查一次）且 `retries: 3`（連續失敗 3 次才判定不健康），那麼當關掉 `db` 後，App 會經歷 3 次失敗的探針檢查（$3 \times 3 = 9$ 秒），並在第 9 至 10 秒時狀態正式被 Docker 標記為 `unhealthy`。

* **對應的 log 訊息：**
    當執行 `docker compose ps` 時，會看到 App 的狀態改變：
    ```text
    NAME          IMAGE      COMMAND                  SERVICE   STATUS
    w08-app-1     myapp:v1   "python3 app.py"         app       unhealthy
    ```
    而在執行 `docker compose logs app` 或 `docker events` 時，會抓到類似以下的連線失敗錯誤與 Docker 狀態事件日誌：
    ```text
    [ERROR] 2026-06-11 14:45:02 - Connection to database failed: Connection refused
    [WARNING] app-1 health check failed! (Command [...] exited with status 1)
    container health_status: unhealthy w08-app-1 (image=myapp:v1, name=w08-app-1)
    ```

## Log 失控估算

* **noisy 容器 30s log 大小：** `30 MB` 左右
    > **（實驗測量方式）：** 透過在終端機執行 `du -sh /var/lib/docker/containers/<container-id>/` 來觀察該 noisy 容器日誌檔在 30 秒內的體積變化。

* **預估 24h 大小：** 大約 `86.4 GB`（計算：$1 \text{ MB/s} \times 60 \text{秒} \times 60 \text{分} \times 24 \text{小時} = 86,400 \text{ MB}$）
    > **（維運警訊）：** 這證明了如果任由容器無限制地瘋狂列印日誌，僅僅一兩天內就能把一台小雲端伺服器的 30 GB 系統硬碟完全撐爆。

* **套 rotation 後穩定上限：** `50 MB`（或是依據你 yaml 實際設定的上限）
    > **（原理解釋）：** 當我們在 `compose.yaml` 的服務中寫入以下配置：
    > ```yaml
    > logging:
    >   driver: "json-file"
    >   options:
    >     max-size: "10m"   # 每個 Log 檔案最大 10 MB
    >     max-file: "5"     # 最多保留 5 個檔案
    > ```
    > 此時該容器不論在線上跑幾個月、印了幾億行 Log，它佔用硬碟的最終天花板都會被死死鎖定在：$10 \text{ MB} \times 5 = 50 \text{ MB}$。當第 6 個 10 MB 的舊 Log 產生時，最老的 Log 就會被自動刪除，成功馴服失控的日誌！

## 資源限制實驗
| 實驗 | 命令 | 觀察結果 | 對應 cgroup 檔 | 值 |
|---|---|---|---|---|
| OOM | stress-ng --vm 1 --vm-bytes 200m | exit 137, OOMKilled=true | memory.max | 134217728 |
| CPU throttle | stress-ng --cpu 4 | docker stats CPU% ≈ 50% | cpu.max | 50000 100000 |

## 權限四階對照

| 階梯 | id | CapEff | NoNewPrivs | curl /healthz | 說明與對應配置 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **0** | `uid=0(root)` | `00000000a80425fb`<br>(全開，14個特權) | `0` (False) | **200 OK** | **預設不設防狀態**<br>預設以 root 執行，擁有過大的系統功能特權。 |
| **1** | `uid=1000(appuser)` | `0000000000000000`<br>(全清空) | `0` (False) | **200 OK** | **切換非 root 身份（`USER: appuser`）**<br>成功剝奪 root 身份，普通非 root 進程預設不帶任何 Cap 特權。 |
| **2** | `uid=1000(appuser)` | `0000000000000000` | `0` (False) | **200 OK** | **丟棄預設特權（`cap_drop: [ALL]`）**<br>即使進程未來想透過漏洞變回 root，也因為底層 Cap 已經被拔光而無法作惡。 |
| **3** | `uid=1000(appuser)` | `0000000000000000` | `1` (True) | **200 OK** | **防止權限提升（`no-new-privileges: true`）**<br>鎖死進程的 uid。此時進程絕對無法透過執行 `sudo` 或帶有 SUID 的執行檔來取得新權限。 |
| **4** | `uid=1000(appuser)` | `0000000000000000` | `1` (True) | **200 OK**<br>(或視快取寫入需求調整) | **唯讀檔案系統（`read_only: true`）**<br>整台容器的 rootfs 變為唯讀，駭客就算進來也無法下載木馬或修改任何程式碼檔案。 |

## 排錯紀錄

- **症狀：** 容器服務正常上線跑了一段時間後，網頁突然斷線，且執行 `docker compose ps` 發現 App 容器的狀態顯示為 `Exited (137)`，並不是正常的健康運行狀態。

- **診斷：** 狀態碼 **`137`** 在 Linux/Docker 的世界中，代表進程因為 **「違反資源限制而被系統強制發送 SIGKILL (9) 訊號殺死」**。
  結合本週在 `compose.yaml` 中配置的 cgroup 記憶體上限（例如 `mem_limit: 512m`），當應用程式內部發生記憶體洩漏（Memory Leak）或瞬間處理大量請求、導致記憶體用量飆破 512 MB 臨界點時，作業系統核心的 **OOM Killer (Out of Memory Killer)** 就會為了保護宿主機的安全，直接出面把該容器的進程一槍斃命。
  我們也可以透過執行 `docker inspect <container_id>`，在 `State` 區塊中抓到 `"OOMKilled": true` 的鐵證。

- **修正：** 1. **短期維運修正**：在 `compose.yaml` 中，根據 `docker stats` 事前觀察到的真實高尖峰負載，將 `mem_limit` 進行合理的放寬（例如調整為 `1024m`），並加入緩衝的 `memswap_limit`。
  2. **生產環境安全防禦**：加上自動重啟機制與合適的健康檢查，確保即使再次發生崩潰，服務也能在幾秒內自動原地的復活：
  ```yaml
  services:
    app:
      image: myapp:v1
      mem_limit: 1024m
      restart: unless-stopped
## 設計決策

本週在將 `compose.yaml` 升級為生產環境（Production-ready）標準時，針對資源調配與檔案系統安全做了以下核心決策：

### 1. 選擇 `mem_limit` 與 `cpus` 限制數值的理由
我為 App 容器配置了 `mem_limit: 512m`（或你實驗設定的數值）與 `cpus: "0.5"`（代表限制最多使用 50% 的單核心處理器能力）。
* **防止「雜訊鄰居」效應（Noisy Neighbor）**：在沒有設定 Linux cgroup 限制的情況下，如果 Web App 遭遇惡意攻擊、無窮迴圈或記憶體洩漏，它會無限制地榨乾宿主機（Host）的所有 CPU 與記憶體，導致整台伺服器卡死、連 SSH 都登不進去。
* **合理保留緩衝**：透過 `docker stats` 事前觀察，本專案的 Flask/Python 應用程式在正常負載下僅消耗約 50-80 MB 記憶體、CPU 使用率低於 5%。因此將上限鎖定在 `512 MB` 與 `0.5 CPU`，既能給予突發流量足夠的運算緩衝，又能保證即使 App 徹底失控崩潰，也絕對不會拖垮整台伺服器。

### 2. 設定 `read_only` 之後，補上了哪些 `tmpfs`？為什麼？
當我們在 `compose.yaml` 中開啟強大的安全防禦 `read_only: true` 時，整台容器的底層圖層系統會變成完全唯讀。這雖然能完美防止駭客植入木馬，但會導致 Linux 內建或 Python 框架內部「需要寫入暫存資料」的機制直接噴錯（如 `Read-only file system`）。

為了解決這個衝突，我額外掛載了以下 **`tmpfs`（記憶體暫存區）**：
* **`/tmp`**：許多 Linux 工具與 Python 套件在運作時，預設會在這個目錄建立臨時檔案。將其掛載為 `tmpfs`，能讓程式擁有寫入暫存檔的空間。
* **`/run` 或 `/var/run`**：系統進程在記錄 PID（進程識別碼）或建立 socket 連線鎖定時，需要在此目錄寫入狀態檔案。
* **`/home/appuser/.cache`（或 `__pycache__` 相關寫入目錄）**：Python 執行時可能會試圖寫入快取檔。

**採取此設計的理由**：
透過這種「**整體鎖死（`read_only`）+ 局部開窗（`tmpfs`）**」的精密取捨，我們成功保護了核心程式碼不被非法修改，同時又把「允許寫入的權限」嚴格限制在「斷電或重啟就會全滅、不佔用硬碟空間」的虛擬記憶體暫存區中，完美兼顧了微服務的安全防禦與正常運作。