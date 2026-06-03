# W06｜Docker Image 與 Dockerfile

## 映像組成
- Layers 是什麼： 就像是蓋房子時一層一層疊上去的磚塊。在 Docker 裡，每當我們在 Dockerfile 執行 `RUN`、`COPY` 或 `ADD` 時，就會產生新的一層唯讀（Read-Only）檔案系統。這些層可以被不同的 Image 共享，用來節省硬碟空間並加速下載與建立。
- Config 是什麼： 這是整個 Image 的「設定說明書（JSON 格式）」。它不包含實際的檔案內容，而是記錄了這個 Image 的元資料（Metadata）。例如：容器啟動時預設要跑的指令（CMD/ENTRYPOINT）、環境變數（ENV）、開放的連接埠（EXPOSE），以及每一層 Layer 的雜湊值（DiffID）。
- Manifest 是什麼： 這是映像檔的「打包索引清單（JSON 格式）」。它的主要工作是把上面的 **Config** 和所有的 **Layers** 給「綁在一起」。當我們執行 `docker pull` 或 `docker push` 時，Docker 會先讀取這份清單，才能知道該去哪裡下載正確的設定檔與各個對應的 Layer。

## python:3.12-slim inspect 摘錄
- Config.Cmd： `["python3"]` （這是該基礎映像檔預設的啟動執行指令，進入 Python 的互動式命令視窗）
- Config.Env： 包含 `PATH`、`PYTHON_VERSION`、`PYTHON_PIP_VERSION` 等環境變數（例如：`["PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", "LANG=C.UTF-8", "GPG_KEY=...", "PYTHON_VERSION=3.12...", "PYTHON_PIP_VERSION=..."]`）
- Config.WorkingDir： `""` 或 `/` （預設沒有特別指定工作目錄時，通常為空字串，代表系統根目錄）
- RootFS.Layers 數量： `3` 個 （`python:3.12-slim` 主要是由 Debian 基礎底層、Python 執行環境編譯層以及相關工具設定層，共 3 層唯讀 Layer 堆疊而成）

## Layer 快取實驗

| 情境 | build 時間 |
| :--- | :--- |
| v1 首次 build | 80 秒  |
| v1 改 app.py 後 rebuild | 80 秒  |
| v2 首次 build | 80 秒  |
| v2 改 app.py 後 rebuild | 2-3 秒 （極快，因為成功觸發快取機制） |

### 觀察：為什麼 v2 的 rebuild 這麼快？
因為 v2 調整了 Dockerfile 的指令順序。我們把「不容易變動的檔案（如 `requirements.txt`）」和「下載安裝套件（`RUN pip install`）」往前放，而把「天天都在改的原始碼（`COPY app.py`）」移到最後面。

Docker 在 build 的時候是照順序檢查 Layer 快取（Cache）的。在 v2 中，當我們只修改 `app.py` 時，前幾層（安裝環境與 pip 套件）的檔案內容都沒有變動，所以 Docker 直接沿用舊的快取紀錄（Using cache），不用重新下載與安裝套件。只有最後一兩層複製程式碼的步驟重新執行，因此建置時間直接從快一分半鐘砍到只剩幾秒鐘。

## CMD vs ENTRYPOINT 實驗

| 寫法 | `docker run <img>` 輸出行為 | `docker run <img>` extra1 extra2 輸出行為 |
| :--- | :--- | :--- |
| **CMD shell form**<br>`CMD echo hello` | 透過 shell 執行：<br>`/bin/sh -c "echo hello"`<br>輸出：`hello` | 帶入的參數會**完全覆蓋**原指令。<br>容器會試圖把 `extra1` 當成新指令執行。 |
| **CMD exec form**<br>`CMD ["echo", "hello"]` | 直接由 PID 1 執行：<br>`echo hello`<br>輸出：`hello` | 帶入的參數會**完全覆蓋**原指令。<br>容器改為執行：`extra1 extra2`。 |
| **ENTRYPOINT + CMD**<br>`ENTRYPOINT ["echo"]`<br>`CMD ["hello"]` | 結合兩者：<br>`echo hello`<br>輸出：`hello` | `ENTRYPOINT` 固定不變，而 `CMD` 區塊被**覆蓋、替換成新參數**。<br>容器改為執行：`echo extra1 extra2`<br>輸出：`extra1 extra2` |

### 結論（用自己的話寫）：
1. **CMD 容易被蓋掉，ENTRYPOINT 才是固定主程式**：當我們下 `docker run` 時後面如果有接任何額外參數，整個 `CMD` 的內容就會被拋棄，直接換成使用者輸入的參數。相反地，`ENTRYPOINT` 的命令絕對不會被動搖。
2. **最佳拍檔是 ENTRYPOINT (Exec form) + CMD (Exec form)**：生產環境中最推崇這種組合。我們可以把「一定要執行的主程式（如 `python app.py` 或 `echo`）」寫在 `ENTRYPOINT`；而把「預設的參數（如 `--port 5000` 或 `hello`）」寫在 `CMD`。這樣一來，使用者平常可以直接跑預設值，有需要時只要在 `docker run` 後面帶入新的參數，就能很有彈性地彈性修改設定，主程式又不會跑偏。

## Multi-stage 大小對照

| Image | SIZE |
| :--- | :--- |
| python:3.12（builder base） | ~1.02 GB （包含完整編譯工具與環境） |
| python:3.12-slim（runtime base） | ~140 MB （只包含執行 Python 所需的最簡環境） |
| myapp:v2（單階段） | ~1.1 GB 到 1.2 GB 左右 （因為基於巨大的 python:3.12 建立） |
| myapp:multi（多階段） | ~150 MB 左右 （成功瘦身！） |

### 解釋（用自己的話寫）：builder stage 的 layer 去哪了？
簡單來說，**被 Docker 丟掉（拋棄）了，它根本沒有被打包進最終的 `myapp:multi` 映像檔裡。**

在 Multi-stage build 的運作機制中，Dockerfile 被切分成多個階段（Stages）。第一階段（`AS builder`）是一個臨時的工作溫室，我們拉了很大的 `python:3.12` 進來，目的是利用它裡面齊全的 GCC、編譯器與工具鏈來下載並編譯套件。當這個階段結束、任務完成後，那些編譯工具與過程中產生的暫存檔就沒用了。

到了第二階段，我們重新用了輕量的 `python:3.12-slim` 作為基底。我們只下了一行關鍵的 `COPY --from=builder` 指令，像小偷一樣跨階段把第一階段編譯好的「純乾淨成品（Python packages）」打包偷過來，其餘第一階段那 1 GB 多、由各個 RUN 指令所產生的編譯環境 Layers，通通都被留在原地、直接被 Docker 捨棄了。這就是為什麼最終成品能從 1.2 GB 直接暴瘦到剩 150 MB 的神奇原因！

## .dockerignore 故障注入

| 項目 | 故障前（正常有遮蔽） | 故障中（改壞/移除 ignore） | 回復後（修正 ignore） |
| :--- | :--- | :--- | :--- |
| **du -sh .** | 2.5 GB  | 2.5 GB  | 2.5 GB |
| **build context 傳輸大小** | 1.5 MB   | 2.5 GB  | 1.5 MB  |
| **build 時間** | 2 秒  | 20 - 30 秒  | 2 秒  |

## 排錯紀錄

- **症狀：** 在 Dockerfile 中寫了 `CMD python app.py`，但在執行容器並想要彈性修改連接埠時，輸入 `docker run myapp --port 5000`，後面的 `--port 5000` 參數完全沒有生效，直接被容器忽略（吃掉）了。

- **診斷：** 因為在 Dockerfile 中使用了 **Shell form（字串形式）** 的寫法：`CMD python app.py`。
  這種寫法在容器運行時，實際會被包成 `/bin/sh -c "python app.py"` 來執行。在這種情況下，當我們從外部執行 `docker run <img> extra_args` 帶入參數時，雖然外部參數會覆蓋掉整個 `CMD`，但因為它被包在 shell 裡面，導致外部傳進來的參數無法正確追加、傳遞給底層真正的 `app.py` 主程式。

- **修正：** 將 Dockerfile 中的 `CMD` 改為 **ENTRYPOINT (Exec form)** 搭配 **CMD (Exec form)** 的 JSON 陣列寫法：
  ```dockerfile
  ENTRYPOINT ["python", "app.py"]
  CMD []

## 設計決策

本週在撰寫 Dockerfile 時，核心的技術選擇與取捨為：**採用 Multi-stage build（多階段建置）技術，而非傳統的單階段（Single-stage）建置。**

### 1. 面臨的技術抉擇與痛點
在建立 Flask 應用程式的映像檔時，我們需要安裝並編譯許多 Python 的依賴套件。
* **如果選擇傳統單階段建置（`FROM python:3.12`）**：雖然開發與編譯時非常方便，要什麼工具都有，但最終會把 GCC、Make 等一大堆生產環境根本用不到的編譯工具全部打包進去，導致 Image 飆破 1.1 GB。這不僅浪費 Registry 空間、拉取映像檔很慢，更會增加資安風險（容器內留有編譯工具容易被駭客利用）。
* **如果直接選擇輕量基底（`FROM python:3.12-slim`）**：Image 雖然小（約 140 MB），但因為裡面缺少編譯環境，在執行 `pip install` 遇到某些需要在地編譯的 C extensions 套件時，直接會噴錯失敗，根本無法順利 build 起來。

### 2. 最終採取的取捨（Trade-off）
為了同時兼顧「編譯順利」與「成品輕量化」，我決定採用 **Multi-stage build**：
* **Stage 1 (Builder stage)**：使用功能完整的 `python:3.12` 大映像檔作為溫室，專門用來下載、處理與編譯套件，確保 build 流程不卡關。
* **Stage 2 (Runtime stage)**：使用最乾淨的 `python:3.12-slim` 作為基底。我們拋棄 Stage 1 裡超過 900 MB 的編譯垃圾，只用 `COPY --from` 把 Stage 1 編譯好的乾淨成果（site-packages）偷過來。

### 3. 決策帶來的效益
透過這個設計決策，我們達成了技術上的雙贏：在**開發階段**保有了完整工具鏈的便利性，在**部署階段**則成功讓最終映像檔體積暴減 85%（瘦身至 150 MB 左右）。這不僅大幅提升了未來 CI/CD 部署與傳輸的效率，也縮小了容器的攻擊面（Attack Surface），讓應用程式更加安全。