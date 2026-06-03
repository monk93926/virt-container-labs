# W07｜Docker Compose 與資料持久化

## 拓樸圖
```text
           +-------------------+
           |       app         |
           |   Flask App       |
           |    port 5000      |
           +---------+---------+
                     |
          default docker network
                     |
                     v
           +-------------------+
           |        db         |
           |    postgres:16    |
           |     port 5432     |
           +---------+---------+
                     |
                     v
              db-data volume
```

## 從 docker run 到 compose.yaml
我覺得最大的改善是可以一次管理多個 container。以前使用 docker run 時，需要自己建立 network、設定環境變數和 volumes，指令很長也容易打錯。使用 compose.yaml 後，可以把所有設定集中管理，只需要 docker compose up 就能同時啟動 app 和 db。

## 三種掛載對照
| 掛載類型 | 路徑（host） | 容器砍重起資料還在嗎 | 重啟容器資料狀態 | 適合情境 |
|---|---|---|---|---|
| named volume |Docker 自己管理的位置 | 會 | 保留 | 資料庫、正式環境 |
| bind mount | 使用者指定的 host 路徑 | 會 | 保留 | 開發環境、同步程式碼 |
| tmpfs | 記憶體中（沒有實體路徑）| 不會 | 消失 | 暫存資料、cache |

## healthcheck 前後對照
| 寫法 | curl /healthz t=1s | t=3s | t=5s | t=10s |
|---|---|---|---|---|
| 只 depends_on | db not ready | db not ready | ok | ok |
| service_healthy | ok | ok | ok | ok |

觀察（自己的話）：只使用 depends_on 時，只能保證 db container 已經啟動，但不代表 PostgreSQL 已經可以接受連線，因此 app 一開始可能連不到資料庫。加入 healthcheck 和 service_healthy 後，Compose 會等 db 真的 healthy 才啟動 app，因此服務比較穩定。

## 排錯紀錄
- 症狀：docker compose up 時出現 yaml control characters are not allowed。
- 診斷：compose.yaml 在貼上時格式損壞，產生錯誤字元。
- 修正：重新建立 compose.yaml，確認縮排與 EOF 正確。
- 驗證：重新執行 docker compose up -d --build 後，app 與 db 都正常啟動。

## 設計決策
（為什麼 db 用 named volume 而不是 bind mount？為什麼不能在生產用 tmpfs 存資料庫？）

我選擇讓 PostgreSQL 使用 named volume，而不是 bind mount。因為 named volume 由 Docker 管理，比較不容易受到 host 路徑權限或檔案結構影響，也比較適合正式環境的資料持久化。
tmpfs 雖然速度很快，但資料只存在記憶體中，container 重啟或主機關機後資料就會消失，因此不適合拿來存放資料庫。
