# 期末實作 — <412631128> <陳冠宏>

## 1. 架構總覽
<img width="928" height="246" alt="螢幕擷取畫面 2026-06-16 050758" src="https://github.com/user-attachments/assets/567adced-d3f7-4a74-93e6-c47f868422b1" />
延續期中 bastion → app 的兩層 VM 架構，Host 端透過 ProxyJump 進入 app VM
，並在 app VM 上以 Docker Compose 管理兩個服務：Flask Web Application 與 PostgreSQL 資料庫。兩者透過內部 network 進行溝通
，其中 App 負責提供 HTTP API（port 8080），DB 提供資料查詢服務。

## 2. Part A：底座與基準點
<ssh 證據 + 版本 + snapshot>

SSH 成功證據
mark@bastion:~$

<img width="650" height="369" alt="螢幕擷取畫面 2026-06-16 025855" src="https://github.com/user-attachments/assets/5e4af31e-f280-4c15-af95-7e68544d8fac" />

版本資訊
docker --version
docker compose version

<img width="637" height="162" alt="螢幕擷取畫面 2026-06-16 030121" src="https://github.com/user-attachments/assets/aeebcd63-7485-45f3-b9a4-2574f3729b92" />

Snapshot

<img width="784" height="617" alt="螢幕擷取畫面 2026-06-16 030451" src="https://github.com/user-attachments/assets/017b660b-fc30-4405-b902-07b66510344e" />

✔ final-baseline（VMware Snapshot 已建立）

## 3. Part B：Dockerfile 與快取
<Dockerfile + 兩次 build 對照>
第一次 build
<img width="679" height="593" alt="螢幕擷取畫面 2026-06-16 033146" src="https://github.com/user-attachments/assets/0204a95a-a4b6-4839-883d-b9345a0218f5" />

第二次 build

<img width="668" height="530" alt="螢幕擷取畫面 2026-06-16 033207" src="https://github.com/user-attachments/assets/63889796-819e-4475-9818-6818787234f3" />

### 為什麼聽 8080 不聽 80？

因為在 Linux 系統中，1024 以下的 port（如 80）屬於 privileged port，需要 root 權限才能綁定。
但本實作 Dockerfile 已使用非 root 使用者（USER appuser），因此無法使用 80 port。
改用 8080 可以在非 root 權限下正常運行，同時符合容器最小權限（least privilege）安全原則

## 4. Part C：Compose 與資料持久化
<compose.yaml 重點 + 三段對照>

資料寫入

<img width="683" height="305" alt="螢幕擷取畫面 2026-06-16 040410" src="https://github.com/user-attachments/assets/4101f32f-6272-42fe-9ee7-240cac37e42e" />

說明：
成功在 PostgreSQL 中建立資料表並寫入資料。

down → up（資料仍存在）

<img width="682" height="411" alt="螢幕擷取畫面 2026-06-16 035650" src="https://github.com/user-attachments/assets/82ecdd21-7dc3-467c-b671-25b2f0b39904" />

說明：
執行 docker compose down 後重新啟動，資料仍然存在，證明 named volume 有生效。

down -v（資料消失）

<img width="683" height="500" alt="螢幕擷取畫面 2026-06-16 041211" src="https://github.com/user-attachments/assets/74b4dfb2-6fc8-478b-979b-40855ce02fe5"/>

說明：
使用 docker compose down -v 刪除 volume，資料被清除。

重新寫入資料

<img width="683" height="367" alt="螢幕擷取畫面 2026-06-16 041539" src="https://github.com/user-attachments/assets/9d5a24ab-47af-4d37-baef-a6e5a4396d00" />

說明：
volume 重建後資料庫恢復可用，重新寫入資料成功。

### down vs down -v

docker compose down：只刪 container，不刪 volume → 資料保留
docker compose down -v：刪 container + volume → 資料完全消失

named volume 生命週期

named volume 不會隨 container 消失，它獨立存在於 Docker 主機上，
只有在執行 down -v 或手動刪除時才會被移除，因此適合用於資料持久化。

## 5. Part D：生產化加固
<權限驗證輸出 + cgroup 讀值對照表>

<img width="643" height="131" alt="螢幕擷取畫面 2026-06-16 022419" src="https://github.com/user-attachments/assets/45d5f7ee-ba0e-419b-9a9f-2d461ceda4c9" />

<img width="643" height="134" alt="螢幕擷取畫面 2026-06-16 023126" src="https://github.com/user-attachments/assets/f3e64e4f-d4de-45fb-8abf-6b19acc38203" />
### yaml 的值怎麼對回 cgroup 檔案？

在 compose.yaml 中設定的資源限制會透過 Linux cgroup 套用到容器。
以本次設定為例，mem_limit: 256m 對應到 268435456 Bytes（256×1024×1024），cpus: "0.5" 對應到 cpu.max 的 50000 100000，
表示容器可使用 50% 的 CPU 時間，而 pids_limit: 200 則直接對應到最多 200 個行程。
因此從 cgroup 讀到的數值可以驗證 compose.yaml 中設定的資源上限已成功套用到系統核心

## 6. Part E：故障演練
### 故障 1：<F1–F4 擇一>
- 注入方式：F1
- 故障前：
- <img width="1639" height="154" alt="螢幕擷取畫面 2026-06-16 044456" src="https://github.com/user-attachments/assets/06a4bb6b-7924-47b2-b7f9-4bab623e0439" />

- 故障中：
- <img width="1639" height="279" alt="螢幕擷取畫面 2026-06-16 044626" src="https://github.com/user-attachments/assets/8eaaa526-1783-4409-9ba3-8eb088d84c5f" />

- 回復後：
- <img width="1646" height="260" alt="螢幕擷取畫面 2026-06-16 044730" src="https://github.com/user-attachments/assets/dd331b02-be29-4751-8171-944d41757be5" />

- 診斷推論：
- 停止資料庫後，app 容器仍然存在並持續執行，但因無法連接 PostgreSQL
- ，healthcheck 失敗而被標記為 unhealthy，因此 unhealthy 不代表容器死亡，而是服務依賴異常。

### 故障 2：<另一個>
- 注入方式：F2
- 故障前：
  
  <img width="643" height="214" alt="螢幕擷取畫面 2026-06-16 044812" src="https://github.com/user-attachments/assets/2ee6470f-1c87-41d7-8a17-9c9b1a0f30e1" />

- 故障中：
- <img width="1636" height="344" alt="螢幕擷取畫面 2026-06-16 044906" src="https://github.com/user-attachments/assets/ff232b09-14b0-4bb4-aea0-2a39c5ba7dc6" />

- 回復後：
- <img width="640" height="134" alt="螢幕擷取畫面 2026-06-16 044958" src="https://github.com/user-attachments/assets/069ccf09-c648-4429-820c-4b4a8e3189e5" />

- 診斷推論：
停止 app 後，主機的 8080 埠不再有服務監聽
，因此 TCP 連線立即被拒絕（connection refused）。這表示問題位於容器或應用程式層，而非網路傳輸層。

### 三症狀分層表（必答）
| 症狀 | 最可能的層 | 第一條驗證命令 |
| timeout|網路層或防火牆層 | curl -v http://目標位址 |
| connection refused | 容器層或應用程式層 |docker compose ps  |
| HTTP 503 | 應用程式依賴服務層 |docker compose logs app  |

## 7. 反思（200 字）
這學期從 VM 做到 production-ready 容器，「隔離」這個概念在 VM、namespace、
cgroup、權限階梯四個地方各出現一次——它們在防的東西一樣嗎？

這學期從 VM 到 production-ready container 的過程中，「隔離」在不同層級以不同形式出現，
但目的並不完全相同。VM 的隔離是硬體層級的完整環境切分，用來確保作業系統彼此獨立
；namespace 則是在作業系統內提供行程、網路與檔案系統的視圖隔離
；cgroup 則負責資源使用的限制與控管，例如 CPU、記憶體與 PID 數量；而權限階梯則是在使用者與 Linux capability 層面限制程式的操作能力。雖然這些機制看似不同，但核心目標都是避免單一程序影響整體系統穩定性與安全性。不同的是，VM 偏向「環境隔離」，namespace 偏向「視圖隔離」，cgroup 偏向「資源隔離」，權限階梯則是「行為隔離」。
四者共同組成現代容器安全與穩定性的基礎，使系統能在共享資源下仍保持可靠運作。
## 8. Bonus（選做）
