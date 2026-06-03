# W06｜Docker Image 與 Dockerfile

## 映像組成
- Layers 是什麼：Layers 是 Docker image 一層一層的檔案變更紀錄，每執行一次 RUN、COPY 等指令都可能產生新的 layer。Docker 可以利用 layer 做快取，減少重新 build 的時間。
- Config 是什麼：Config 是 image 的設定資訊，包含環境變數、預設執行指令、工作目錄等內容，主要是告訴 Docker 容器啟動時要怎麼執行。
- Manifest 是什麼：Manifest 是 image 的清單，裡面記錄 image 包含哪些 layers、config 位置以及 image 的相關資訊，Docker 會依照 manifest 去組合完整的 image。

## python:3.12-slim inspect 摘錄
- Config.Cmd：["python3"]
- Config.Env：["PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin","LANG=C.UTF-8","PYTHON_VERSION=3.12"]
- Config.WorkingDir：/
- RootFS.Layers 數量：4

## Layer 快取實驗
| 情境 | build 時間 |
|---|---|
| v1 首次 build | 1m7.069s |
| v1 改 app.py 後 rebuild | 0m22.916s |
| v2 首次 build | 0m0.130s |
| v2 改 app.py 後 rebuild | 0m11.262s |

觀察（用自己的話寫）：為什麼 v2 的 rebuild 這麼快？
v2 的 rebuild 比較快，是因為 Dockerfile 先處理 requirements.txt 和 pip install，最後才 COPY app.py。這次我只改 app.py，requirements.txt 沒有變，所以前面安裝套件的 layer 可以直接使用快取，不用重新下載和安裝套件。Docker 只需要重新執行 app.py 之後的那幾層，因此 build 時間變短。
## CMD vs ENTRYPOINT 實驗
| 寫法 | `docker run <img>` 輸出 | `docker run <img> extra1 extra2` 輸出 |
|---|---|---|
| CMD shell form | hello shell form | executable file not found |
| CMD exec form | hello exec form | executable file not found |
| ENTRYPOINT + CMD | hello entrypoint | extra1 extra2 |

結論（用自己的話寫）：CMD 比較像是容器的預設指令，使用者在 docker run 後面加參數時，原本的 CMD 會被覆蓋掉。ENTRYPOINT 則是固定會執行的主程式，即使 docker run 後面有額外參數，也只是把那些參數加到 ENTRYPOINT 後面。因此 ENTRYPOINT 比較適合用來固定容器的主要功能，而 CMD 比較適合提供預設參數。

## Multi-stage 大小對照
| Image | SIZE |
|---|---|
| python:3.12（builder base） | 1.62GB |
| python:3.12-slim（runtime base） | 179MB |
| myapp:v2（單階段） | 1.63GB |
| myapp:multi（多階段） | 189MB |

解釋（用自己的話寫）：builder stage 的 layer 去哪了？
builder stage 的 layer 沒有被放進最後的 image 裡。Multi-stage build 只會保留最後一個 stage 的 layer，前面的 builder stage 只是拿來安裝套件、編譯或建置檔案，最後只把需要的檔案 COPY 到 runtime stage。因此 builder stage 本身的 layer 會被捨棄，不會包含在最後的 image 中。
## .dockerignore 故障注入
| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| du -sh . | 501M | 501M | 501M |
| build context 傳輸大小 | 5.12kB | 524.3MB | 5.12kB |
| build 時間 | 0m5.913s | 0m1.655s | 0m0.244s |

## 排錯紀錄
- 症狀：Docker build 時 build context 大小突然變很大。
- 診斷：檢查後發現 .dockerignore 被刪掉，導致 bigdata 裡的 500MB 大檔案也被送進 build context。
- 修正：重新建立 .dockerignore，並加入 bigdata、.git、__pycache__、*.pyc 等排除項目。
- 驗證：重新 build 後，build context 從 524.3MB 降回 5.12kB。

## 設計決策
（說明本週至少 1 個技術選擇與取捨，例如：為什麼 runtime 選 `python:3.12-slim` 而不是 `alpine`？）

本週我選擇使用 python:3.12-slim 當 runtime image，而不是 alpine。
原因是 python:3.12-slim 的體積比完整的 python:3.12 小很多，可以減少 image 大小；同時它又比 alpine 更接近一般 Linux 環境，Python 套件相容性比較好。alpine 雖然更小，但有些套件可能需要額外編譯或安裝相依套件，排錯成本比較高。
所以這次的取捨是：不追求最小體積，而是選擇體積較小、相容性也比較穩定的 python:3.12-slim。
