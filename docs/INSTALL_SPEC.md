# install.sh 設計規範

## 核心設計哲學：驗證重要性 > 安裝本身

**背景**：case01-20260527 的 install.sh 從 v1 迭代到 v8，根本原因不是邏輯錯誤，而是每次失敗時資訊不足，只能猜測後修改再試。迭代次數多的本質是「觀測不夠」，不是「邏輯不好」。

**原則**：install.sh 的首要任務是**讓問題在第一時間就看得清楚**，而不是讓安裝盡快完成。多印一倍的資訊，比多跑一次安裝更有價值。

---

## A. 安裝前：必須完成的驗證（gate，不通過不能繼續）

**A1. 印出完整的安裝清單**
- 每個步驟安裝前，列出所有將安裝的套件名稱（含 --nodeps 群組獨立列出）
- 不能只印「共 N 個 RPM」，必須逐一列名
- 目的：人眼一掃就能發現遺漏或多餘的套件

```
[INFO] -- 套件選擇（共 19 個）--
[INFO] [--nodeps 群組 2 個]
[INFO]   dkms-3.4.1-1.el9.noarch.rpm        [理由: kernel-devel-matched]
[INFO]   vulkan-loader-1.4.x.rpm            [理由: mesa-vulkan-drivers 衝突]
[INFO] [正常安裝 17 個]
[INFO]   nvidia-driver-610.43.02-1.el9.rpm
...
```

**A2. rpm --test 是 gate，不是 log**
- 執行 `rpm --test --force` 後，分析輸出：
  - 有 NVIDIA/CUDA 相關缺失依賴 → **停止，回報，不繼續安裝**
  - 只有系統/桌面套件缺失 → 記錄警告，繼續
- 不能只把 --test 結果印到 log 就繼續（v1~v4 的問題）

**A3. --nodeps 套件必須附理由**
- 每個 --nodeps 安裝都必須在 log 中說明為什麼需要跳過依賴檢查
- 沒有理由的 --nodeps 是風險，不允許出現

---

## B. 安裝中：足夠的可觀測性

- --nodeps 套件**逐一安裝**，不批次，每個都有獨立輸出
- 批次安裝的 rpm 輸出**完整保留**在 log（不截斷 progress bar）
- 每個步驟開始和結束都有時間戳

## B1. log 檔案命名規則：序號制，不覆蓋

每次執行 install.sh 產生新的 log，不覆蓋舊的：

```
install_001.log   ← 第一次執行
install_002.log   ← 第二次執行（修正後重試）
install_003.log   ← 第三次執行
...
```

腳本啟動時自動偵測現有最大序號，+1 命名新 log：
```bash
LOG_NUM=$(ls install_[0-9]*.log 2>/dev/null | grep -oE '[0-9]+' | sort -n | tail -1)
LOG_NUM=$(printf '%03d' $(( ${LOG_NUM:-0} + 1 )))
LOG_FILE="install_${LOG_NUM}.log"
```

帶出目標機時只需帶出最新的 `install_XXX.log`，即可完整診斷。

---

## C. 每個步驟結束後立即驗證，結果寫入 log，無論通過與否都繼續執行

安裝流程不是「五個步驟裝完再一起驗證」，而是：

```
步驟1 安裝 → 步驟1 驗證（記錄結果）→ 步驟2 安裝 → 步驟2 驗證（記錄結果）→ ... → 步驟5 驗證（記錄結果）
```

**驗證的目的是觀測，不是 gate。** 驗證不通過仍然繼續跑完所有步驟，不中斷。

原因：拿到 log 後能一眼看出哪個步驟開始有缺漏，哪些步驟是正常的。
若驗證失敗就停止，後面步驟的狀況就看不到，反而增加診斷難度。

不能只靠 rpm 的 exit code 判斷成功，必須主動驗證，且**驗證結果（含失敗）必須完整寫入 log**。

每個步驟的驗證結果格式：
```
[OK]  [STEP 2 驗證] nvidia-driver          : 610.43.02
[OK]  [STEP 2 驗證] nvidia-driver-cuda     : 610.43.02
[OK]  [STEP 2 驗證] dkms status            : nvidia/610.43.02 installed
[OK]  [STEP 2 驗證] /usr/bin/nvidia-smi    : 存在
[FAIL][STEP 2 驗證] nvidia-driver-libs     : 未安裝  ← 這裡就能看出缺漏
```

各步驟驗證清單：

| 步驟 | 必須驗證的項目 |
|------|--------------|
| STEP 1 kernel | gcc 版本、make 版本、kernel-devel 版本與執行中 kernel 一致 |
| STEP 2 driver | rpm -q nvidia-driver、nvidia-driver-cuda、nvidia-driver-libs、dkms status（需顯示 installed）、/usr/bin/nvidia-smi 存在 |
| STEP 3 CUDA | rpm -q cuda-cudart-XX-X、nvcc 存在（/usr/local/cuda/bin/nvcc）|
| STEP 4 cuDNN | rpm -q libcudnn*-cuda-XX、libcudnn*.so 可被載入 |
| STEP 5 Container TK | nvidia-ctk --version 可執行、/etc/docker/daemon.json 含 nvidia runtime |

**DKMS 特別規定**：
- 安裝後必須確認 `dkms status` 顯示 `installed`，不能只看 rpm 安裝成功
- 如果 dkms status 為空，代表 kernel module 沒有建立，是失敗狀態
- dkms status 輸出必須完整寫入 log

---

## D. 失敗時：錯誤資訊與狀態快照必須印入 log

寫入時機：
- 每個步驟的驗證完成後 → 將當時的 state snapshot 印入 log
- 發生錯誤時 → 將完整錯誤內容印入 log

log 中的格式：
```
[2026-01-01 10:00:00] === STATE SNAPSHOT ===
  kernel_packages   : success
  nvidia_driver     : failed
  cuda_toolkit      : not_started
  ...
[2026-01-01 10:00:00] === ERROR DETAIL ===
  failed_step  : nvidia_driver
  error_message: rpm 安裝失敗：mesa-vulkan-drivers is needed by vulkan-loader
  missing_deps : mesa-vulkan-drivers(x86-64)
```

錯誤內容必須包含：
- `failed_step`：哪個步驟
- `error_message`：具體錯誤（不能只寫「請查閱 log」）
- 缺少的依賴清單（如果是依賴問題）

---

## E. 白名單設計原則

- 使用 `case` 語句（bash glob），不用 `grep -E`（環境相容性問題）
- 模式用 `name*` 不用 `name-*`（rpm %{NAME} 返回裸名）
- --nodeps 群組和正常安裝群組**分開管理**，不能混用
- 白名單的知識來源在 `scripts/validate/notes/whitelist.md`

---

## 原有規範

- 安裝前先確認目標機環境與 `sysinfo.json` 一致
- 已安裝的套件跳過（避免衝突）
- 有舊版 driver 先移除再安裝
- 斷點續裝：每個步驟的執行狀態由腳本內部維護，重新執行時自動跳過已完成的步驟
