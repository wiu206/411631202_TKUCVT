# W05｜把容器拆開來看：Namespace / Cgroups / Union FS / OCI

## Docker 環境

- Storage Driver：overlayfs
- Cgroup Version：systemd
- Cgroup Driver：2
- Default Runtime：runc

## Namespace 觀察

### 六種 namespace 用途（用自己的話）
- PID：隔離行程編號。讓容器以為自己是系統上唯一的行程，在容器內下 ps 指令只會看到自己的 Process（通常 PID 是 1），看不到 Host 上的其他行程。
- NET：隔離網路環境。讓容器擁有自己獨立的網卡（例如 eth0、lo）、IP 位址、路由表和 Port 號，不會跟 Host 或其他容器的網路衝突。
- MNT：隔離檔案系統掛載點。讓容器以為自己有一套獨立的根目錄（/），在容器裡掛載或卸載磁碟，不會影響到 Host 原本的檔案系統。
- UTS：隔離主機名稱（Hostname）。讓容器可以設定自己專屬的名字，不用跟 Host 共用同一個主機名稱。
- IPC：隔離行程間通訊。確保只有同一個容器內的 Process 可以互相傳遞訊息（例如 shared memory、號誌等），不會不小心去干擾到 Host 或其他容器的 Process 通訊。
- USER：隔離使用者和群組 ID。可以讓容器內的使用者自以為是 root（擁有最高權限），但實際上在 Host 系統中，他只是一個沒有特權的普通使用者，藉此提升安全性。

### Host vs 容器 inode 對照
| Namespace | Host PID 1 inode | 容器 sleep inode | 一樣嗎？ |
| :--- | :--- | :--- | :--- |
| pid | 4026531836 | 4026532613 | No |
| net | 4026531833 | 4026532615 | No |
| mnt | 4026531832 | 4026532606 | No |
| uts | 4026531838 | 4026532611 | No |
| ipc | 4026531839 | 4026532612 | No |
| user | 4026531837 | 4026531837 | Yes |

### 容器內 `ps aux` 輸出
看見的行程數量： 2 支。

為什麼？ 因為這個容器被隔離在獨立的 PID Namespace 裡。Linux 核心向該容器隱藏了 Host 上的其他行程視角，並將容器的啟動主行程（sleep 3600）重新編號為 PID 1。在這個小世界裡，它只能看到自己以及被臨時派生出來的 ps aux（PID 7），完全碰不到外面的行程。
## Cgroups 實驗

### 容器內讀到的限制
- memory.max：268435456
- cpu.max：50000 100000

### Host 端對照（用 `docker inspect -f '{{.HostConfig.CgroupParent}}'` 動態取得路徑）
- memory.max：268435456
- cpu.max：50000 100000
- memory.current（執行時某一刻）：401408

### OOM 故障三階段
| 項目 | 故障前 | 故障中（memory=32m + dd 200m）| 回復後（memory=256m）|
|---|---|---|---|
| 容器 exit code | - | 137| 0|
| OOMKilled | - | true| false|
| dmesg 關鍵字 | 無 OOM | oom-kill 或 Killed process| 無 OOM |

## Image 分層

### `docker image inspect nginx:1.27-alpine` layer 數量
1
### 兩個同源 image 共享 layer 的證據
**是，完全相同。**

我比對了原生 `alpine` 映像檔與透過它 commit 產生的 `alpine-child` 映像檔，兩者在 `RootFS.Layers` 中最底層的雜湊值皆為 `sha256:989e799e634906e94dc9a5ee2ee26fc92ad260522990f26707861a5f52bf64e`。這證實了 Union FS 的分層共享機制，同源的映像檔會共用相同的唯讀層以節省實體磁碟空間。
### `docker diff` 輸出範例與解讀
### docker diff 輸出範例
C /etc
C /etc/passwd
D /etc/fstab
C /root
A /root/hello.txt

### A/C/D 實例說明
這是 Union FS 的「可寫層 (Writable Layer)」捕捉到的動態變化：
* **A (Add) 實例：** `/root/hello.txt`。當我們在容器內新增原本不存在的檔案時，該檔案會直接寫入最上層的可寫層。
* **C (Change) 實例：** `/etc/passwd`。當我們修改映像檔原本就有的檔案時，底層會觸發 Copy-on-Write (CoW) 機制，把唯讀層的檔案複製一份到可寫層進行修改，而隱藏原本的唯讀檔。
* **D (Delete) 實例：** `/etc/fstab`。當我們刪除映像檔原本內建的檔案時，Docker 並不會真的去動唯讀層的資料（因為唯讀層不可變），而是透過可寫層建立一個特殊的遮罩（Whiteout 檔案）將其「蓋住」，讓容器視角以為它不見了。
## OCI 呼叫鏈

### 1. OCI 呼叫鏈分工（用自己的話說明）
當我們敲下 `docker run` 指令到容器真正跑起來，底層經歷了以下組件的接力合作：

* **dockerd (總管)：** 負責最上層面對使用者的業務，處理 Docker API、高階網路（Network）與儲存卷（Volume）管理，它不直接操作容器，而是負責發號施令。
* **containerd (大班長)：** 負責高階容器執行期管理。收到 dockerd 的命令後，負責把映像檔解壓縮準備好，並根據需求生成一份符合 OCI 標準的 `config.json` 設定檔。
* **containerd-shim (專屬保母)：** 每跑一個容器就會有一個專屬的 shim。它的工作是留在背景默默陪伴容器，負責接管容器的輸出入（I/O）、收集退出碼，並確保 containerd 重啟時容器不會死掉。
* **runc (執行官)：** 負責低階容器執行期。它完全遵循 OCI 規範，工作單一且短暫：讀取 containerd 給它的 `config.json`，直接向 Linux Kernel 申請切出 Namespace、綁上 Cgroups 把容器建立起來，任務完成後立刻退出。

---

### 2. OCI Runtime Spec `config.json` 欄位對應
runc 是根據 `config.json` 裡的宣告來對 Linux 核心下指令，其關鍵對應欄位為：
* **Namespace 對應欄位：`linux.namespaces`**
  這是一個陣列區塊，裡面宣告了容器需要開啟哪些隔離空間（例如 `{"type": "pid"}`、`{"type": "network"}`），runc 看到後就會去呼叫 Kernel 切分視角。
* **Cgroup 對應欄位：`linux.resources`**
  這個區塊宣告了資源限制的實體數值，裡面包含如 `memory`、`cpu` 等子項目，runc 會依此將限制寫入 `/sys/fs/cgroup/` 檔案中。）

## 排錯紀錄
* **症狀：**
  執行 `docker run`、`docker info` 或 `docker inspect` 等指令時，系統噴出 `permission denied while trying to connect to the docker API at unix:///var/run/docker.sock` 的錯誤訊息，導致無法順利操作 Docker。

* **診斷：**
  Docker daemon 的通訊端點 `/var/run/docker.sock` 預設擁有者是 `root`，且僅允許屬於 `docker` 群組的使用者進行讀寫。由於目前的非 root 使用者（如 woody）尚未成功載入 `docker` 群組的權限，因此在沒有加上 `sudo` 的情況下會被核心的安全機制拒絕連線。

* **修正：**
  1. 使用 `sudo usermod -aG docker $USER` 指令，將目前的使用者正式加入 `docker` 群組中。
  2. 執行 `newgrp docker` 指令，讓當前的終端機會話（Session）立刻載入並套用新的群組權限，免去登出再登入的麻煩。

* **驗證：**
  再次於終端機執行不帶 `sudo` 的指令（如 `docker info` 或 `docker run`），指令成功順利通行且無任何報錯，證實權限設定已正確修正完畢。

## 想一想（回答 3 題）
### 1. 容器裡的 PID 1 跟 host PID 1 是同一支 process 嗎？`kill -9 1`（在容器內）會發生什麼？

* **是否為同一支：** **不是。** Host 的 PID 1 是實體機或虛擬機開機時的第一個行程（通常是 `systemd` 或 `init`）；而容器內的 PID 1 只是透過 **PID Namespace** 重新編號、映射過後的容器主行程（例如 `sleep 3600` 或 `nginx`）。它們在 Host 的視角裡，其實是完全獨立且擁有不同真實 PID 的兩個行程。
* **執行 `kill -9 1` 會發生什麼：** Linux 核心對 PID 1 有特殊的保護機制。在容器內執行 `kill -9 1` 時，核心預設會忽略這個強制擊殺訊號（不予理會），以防止容器無故崩潰。除非該主行程自己寫了捕捉訊號並退出的邏輯，否則通常**什麼事都不會發生**；但如果成功殺死它，由於容器是「圍繞著 PID 1 行程生存」的，**PID 1 一旦退出，整個容器就會立刻終止並結束運行**。

---

### 2. 兩個容器都基於 `ubuntu:24.04`，磁碟空間是吃兩份還是共用？怎麼驗證？

* **空間佔用：** **共用同一份。**
* **原理說明：** Docker 底層採用了 **Union FS（聯合檔案系統）**。當兩個容器都基於同一個基礎映像檔（Base Image）時，它們在磁碟上會百分之百共享同一個唯讀層（Read-Only Layer）。只有當容器在運行中修改或新增檔案時，才會透過 Copy-on-Write (CoW) 機制寫入各自獨立、極小的「可寫層（Writable Layer）」，因此不會吃掉兩份 Ubuntu 的磁碟空間。
* **驗證方式：** 透過指令 `docker image inspect ubuntu:24.04` 查看其 `RootFS.Layers` 欄位中的 SHA256 雜湊值（指紋）。接著再去檢視這兩個容器的 `docker inspect <容器名稱>` 輸出，會發現它們指向的底層 Image ID 與分層雜湊值指紋**完全一模一樣**，這就是它們在磁碟上實體共用同一個區塊的鐵證。

---

### 3. 如果 host 的 kernel 爆漏洞，容器還能稱為「隔離」嗎？這個限制跟 VM 差在哪？

* **還能稱為隔離嗎：** **無法稱為完全隔離（安全邊界會破裂）。**
* **限制與差異解析：** 容器本質上**沒有自己獨立的作業系統核心**，它們只是 Host Kernel 上跑著的普通行程，全靠 Namespace 和 Cgroups 圍起來。如果 Host Kernel 爆出權限提升（Privilege Escalation）或逃逸漏洞，容器內的惡意行程就能利用該漏洞直接穿透邊界，控制整個 Host 甚至是其他容器。
* **跟 VM 的差別：**
    * **VM（虛擬化硬體）：** 每個 VM 都有自己**完全獨立且完整的一套 Guest OS Kernel**。VM 裡的行程如果想攻擊 Host，必須先打穿自己的 Kernel，再打穿 Hypervisor（虛擬化軟體層），攻擊面極小，隔離性極強。
    * **Container（虛擬化行程視角）：** 所有容器與 Host 擠在**同一個 Kernel** 上。一旦這個核心倒了，大家就一起中標。這就是容器在追求「輕量、好啟動」時，在安全防護上所做出的先天權衡與限制。