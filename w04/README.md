# W04｜Linux 系統基礎：檔案系統、權限、程序與服務管理

## FHS 路徑表

| FHS 路徑 | FHS 定義 | Docker 用途 |
|---|---|---|
| /etc/docker/ | 系統設定檔目錄 | 存放 Docker daemon 設定（daemon.json） |
| /var/lib/docker/ | 可變資料目錄 | 儲存 image、container、volume 等資料 |
| /usr/bin/docker | 可執行檔位置 | Docker CLI 指令 |
| /run/docker.sock | 執行時期 socket 檔案| CLI 與 Docker daemon 溝通 |

## Docker 系統資訊

- Storage Driver：（<img width="255" height="24" alt="image" src="https://github.com/user-attachments/assets/be28dbe1-ca31-4b44-81a1-01d619497601" />
）
- Docker Root Dir：（<img width="326" height="28" alt="image" src="https://github.com/user-attachments/assets/69969f69-0d19-4b04-b58c-2ffe4db91ce2" />
）
- 拉取映像前 /var/lib/docker/ 大小：（660K）
- 拉取映像後 /var/lib/docker/ 大小：（165M）

## 權限結構

### Docker Socket 權限解讀
（<img width="816" height="143" alt="image" src="https://github.com/user-attachments/assets/3b15ab19-6e93-40a7-b8bf-7b1f5cc52bae" />
owner（root）：可讀寫
group（docker）：可讀寫
others：沒有權限
s：代表 socket 檔案）

### 使用者群組
（<img width="816" height="143" alt="image" src="https://github.com/user-attachments/assets/9f4cc990-804d-45ac-aef5-18d999e2ce68" />
由輸出結果可知，目前使用者已加入 docker 群組（groups 中包含 docker），因此可以直接使用 docker 指令，而不需要透過 sudo 執行。

這也說明 docker.sock 的存取權限（group 為 docker）已正確套用至此使用者。）

### 安全意涵
（用自己的話說明為什麼 docker group ≈ root，安全示範的觀察結果）

## 程序與服務管理

### systemctl status docker
（<img width="810" height="422" alt="image" src="https://github.com/user-attachments/assets/b7cbca7f-a322-4f49-8db9-b7830af2d81c" />
）

### journalctl 日誌分析
（<img width="829" height="579" alt="image" src="https://github.com/user-attachments/assets/ada1bc38-ad4a-4f62-8af9-eb2e72cd8a12" />
從日誌可觀察到 Docker 服務於指定時間啟動，systemd 成功啟動 docker.service，並由 dockerd 進行初始化處理。

日誌中未出現錯誤訊息，表示 Docker daemon 啟動過程正常，系統運作穩定。）

### CLI vs Daemon 差異
（CLI 是使用者輸入指令的工具，Daemon 是實際執行 Docker 的服務。

docker --version 只會檢查 CLI 是否存在，不代表 Docker daemon 有啟動，因此即使版本正常，docker ps 仍可能無法使用。）

## 環境變數

- $PATH：（<img width="806" height="153" alt="image" src="https://github.com/user-attachments/assets/c65fa720-5f19-4d6b-895c-8d16e689ecca" />
）
- which docker：（/usr/bin/docker）
- 容器內外環境變數差異觀察：（主機環境變數較完整，而容器內通常較精簡，僅包含必要路徑。  
容器與主機的環境變數彼此獨立，不會互相影響。）

## 故障場景一：停止 Docker Daemon

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| systemctl status docker | active | inactive (dead) | active |
| docker --version | 正常 | 正常  | 正常 |
| docker ps | 正常 | Cannot connect | 正常 |
| ps aux grep dockerd | 有 process |  無 process |有 |

## 故障場景二：破壞 Socket 權限

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ls -la docker.sock 權限 | srw-rw---- | ---------- | srw-rw---- |
| docker ps（不加 sudo） | 正常 | permission denied | 正常 |
| sudo docker ps | 正常 | 正常 | 正常 |
| systemctl status docker | active | active | active |

## 錯誤訊息比較

| 錯誤訊息 | 根因 | 診斷方向 |
|---|---|---|
| Cannot connect to the Docker daemon |  Docker daemon 未啟動或未運行 | 檢查 systemctl status docker，確認服務是否為 active |
| permission denied…docker.sock | permission denied…docker.sock | 使用者沒有存取 docker.sock 的權限 | 檢查 socket 權限與使用者是否在 docker 群組 |

（用自己的話說明兩種錯誤的差異，各自指向什麼排錯方向）

## 排錯紀錄
- 症狀：執行 docker ps 時出現 permission denied while trying to connect to the docker API 錯誤，無法使用 Docker 指令。
- 診斷：先檢查 /var/run/docker.sock 權限（ls -la），發現權限被設為 000，且一般使用者無法存取。
- 修正：使用 sudo chmod 660 /var/run/docker.sock 將權限恢復為 root 與 docker 群組可讀寫。
- 驗證：重新執行 docker ps 指令後可正常顯示 container，確認問題已排除。

## 設計決策
本次選擇將使用者加入 docker 群組（usermod -aG docker），而不是每次執行 docker 指令時都使用 sudo。

原因在於加入 docker 群組後，使用者可以直接操作 Docker，提升操作效率並減少重複輸入 sudo 的不便，對於教學與實驗環境較為方便。

然而此做法也存在風險，因為 docker 群組的權限等同於 root，使用者可以透過 container 存取主機系統資源，若帳號遭入侵，可能造成系統安全問題。

因此此方法適用於教學或測試環境，但在正式環境中應謹慎使用，並搭配權限控管與安全機制。
