# W08｜容器生產實踐

## Healthcheck 故障測試
- 停 db 後幾秒被標 unhealthy:20秒
- 對應的 log 訊息：health check failed
connection refused

## Log 失控估算
- noisy 容器 30s log 大小：約 3 MB
- 預估 24h 大小：約 8–9 GB
- 套 rotation 後穩定上限：約 30 MB

## 資源限制實驗
| 實驗 | 命令 | 觀察結果 | 對應 cgroup 檔 | 值 |

| OOM |stress-ng --vm 1 --vm-bytes 200m|exit 137，OOMKilled=true，容器被系統強制終止|memory.max|134217728 (128MB)|

|CPU throttle|stress-ng --cpu 4|CPU 使用率被限制，docker stats 約 40%~60%|cpu.max|150000 100000)|

| OOM | stress-ng --vm 1 --vm-bytes 200m | exit 137, OOMKilled=true | memory.max | 134217728 |

| CPU throttle | stress-ng --cpu 4 | docker stats CPU% ≈ 50% | cpu.max | 50000 100000 |

## 權限四階對照
| 階梯 | id | CapEff | NoNewPrivs | curl /healthz |
|---|---|---|---|---|
| 0 | uid=0(root) gid=0(root) | full capabilities | 0 | success |
| 1 | uid=1000(app) gid=1000(app) | partial capabilities | 0 | success |
| 2 | uid=1000(app) gid=1000(app) | reduced capabilities | 1 | success |
| 3 | uid=1000(app) gid=1000(app) | minimal capabilities | 1 | success |
| 4 | uid=1000(app) gid=1000(app) | cap_drop ALL | 1 | success / limited |

## 排錯紀錄
- 症狀 / 診斷 / 修正 / 驗證

症狀：
容器啟動後 healthcheck 持續失敗，狀態顯示 unhealthy，API /healthz 無法正常回應。

診斷：
使用 docker inspect 檢查 health log，發現 connection refused，判斷為後端 DB 容器未啟動或未正常連線，導致應用服務無法通過 healthcheck。

修正：
重新啟動 DB 容器，確認 network bridge 正常，並檢查 service 設定中的 DB host 與 port 是否正確。

驗證：
使用 docker ps 確認 health status 回復 healthy，並測試 API 回應正常。

## 設計決策
（你選的 mem_limit / cpus 數值理由是什麼？read_only 之後你補了哪些 tmpfs，為什麼？）
mem_limit 設為 128m 是為了測試 OOM 機制；cpus 設為 0.5 用來觀察 CPU throttling 行為。
啟用 read_only 後補上 /tmp 與 /run 的 tmpfs，分別提供暫存與 runtime 所需的可寫空間，確保應用在唯讀環境下仍能正常運作。
