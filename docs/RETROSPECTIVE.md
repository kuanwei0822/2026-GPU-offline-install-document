# 作業檢討紀錄

每次作業完成後，記錄實際過程、遭遇的問題、與 Claude 的討論結論，以及帶入流程的改善。
這份文件的價值在於：未來遇到類似問題時，能快速找到「當時怎麼解決的」和「為什麼這樣改」。

---

## 2026-05-28｜case01-20260527 Rocky Linux 9.6 × Blackwell GPU

### 作業目標
在目標伺服器（Rocky Linux 9.6）上離線安裝：
- NVIDIA Driver 610.43.02（open module，Blackwell 需要）
- CUDA Toolkit 13.0
- cuDNN 9.22
- NVIDIA Container Toolkit 1.19.1

---

### 實際執行過程

| 時間 | 動作 | 結果 |
|------|------|------|
| 上午 | 執行 install_v1.sh | ❌ 失敗 |
| | 帶出 install.log，請 Claude 分析 | |
| | 執行 install_v2.sh | ❌ 失敗 |
| | 執行 install_v3.sh | ❌ 失敗 |
| | 執行 install_v4.sh | ❌ 失敗 |
| | 請 Claude 問診斷指令，執行 temp1.sh 結果給 Claude | ❌ temp1.sh 本身有 grep 相容性問題 |
| | 執行 install_v5.sh | ❌ 失敗 |
| | 要求 Claude 改善 log 資訊（v5 資訊太少，難以判斷）| |
| | 執行 install_v6.sh | ❌ 失敗，但跑了很多步驟，明顯進步 |
| | 執行 install_v7.sh | ❌ 失敗 |
| | 執行 install_v8.sh | ✅ install.sh 跑完 |
| 15:00 | sudo reboot | |
| 15:07 | 開機完成，執行驗證 | ❌ nvidia-smi: command not found |
| | 問 Claude，Claude 未主動提供診斷指令，由執行人主動要求 | |
| | 補下載 packages2（須在目標機本機下載，有網路限制）| |
| | 執行 download_open_module.sh（第一次）| ❌ 失敗 |
| | 執行 download_open_module.sh（第二次）| ❌ 失敗，請 Claude 確認而非繼續 try |
| | 執行 download_open_module.sh（第三次）| 與 Claude 排查，改用 wget 直接下載 |
| | 執行 install_v9.sh | 補裝，但有問題（DKMS 被誤刪）|
| 15:47 | sudo reboot | |
| 15:51 | 開機完成，仍有問題 | |
| | 請 Claude 給診斷指令，完整確認狀態 | |
| | 放棄使用 .sh，改由 Claude 直接給 rpm 指令逐步補裝 | |
| | 嚴格一再確認後補裝完成 | |
| 16:30 | sudo reboot | |
| ~16:35 | 開機完成，全部驗證通過 ✅ | |

---

### 問題根因分析

#### 根因1（最重要）：步驟2未識別 GPU 架構
sysinfo.json 中明確有 `GB202GL Blackwell`，但步驟2驗證未標記「此 GPU 需要 open kernel module，下載策略必須調整」。這導致後續用 proprietary module 安裝，全部徒勞。

- **影響**：install_v1～v8 共 8 次失敗，均可避免
- **改善**：步驟2驗證加入 GPU 架構識別，步驟4靜態驗證加入 CHECK 8 為必要項

#### 根因2：install.sh log 資訊不足
每次失敗只能看到「rpm 安裝失敗」，無法直接看出哪個套件缺什麼依賴，只能猜。

- **影響**：每次修一個問題，再跑，再發現下一個問題，形成逐步迭代而非一次解決
- **改善**：install.sh 必須列出完整套件清單、rpm --test 結果、每步驟驗證

#### 根因3：重開機後無明確驗證流程
步驟6說「安裝成功 → 結束」，但實際「腳本跑完」≠「GPU 可用」，重開機後的驗證沒有對應的處理路徑。

- **影響**：nvidia-smi 失敗時，執行人不知道流程應該怎麼走
- **改善**：步驟6拆出「重開機後驗證」子步驟，失敗時明確進入情況D

#### 根因4：Claude 遇到失敗繼續嘗試，而非先確認
download_open_module.sh 多次失敗，Claude 修改腳本繼續嘗試，而非停下來讓執行人確認狀態。

- **影響**：浪費時間，執行人需要主動制止
- **改善**：Claude 行為規範：第一次失敗後確認狀態，不盲目修改再試

#### 根因5：情況D缺乏明確觸發條件
install.sh 出現問題後，執行人自己判斷「放棄腳本，改為互動模式」，流程沒有指引何時應切換。

- **影響**：切換時間點太晚，之前多次無效嘗試
- **改善**：install.sh 第二次失敗 → 強制進入情況D

---

### 討論後帶入流程的改善

| # | 改善項目 | 影響的文件 |
|---|---------|-----------|
| A | 步驟6新增「重開機後驗證」子步驟，失敗 → 情況D | `docs/WORKFLOW.md` |
| B | install.sh 第二次失敗自動觸發情況D | `docs/WORKFLOW.md` |
| C | Claude 行為規範：失敗時先確認狀態，不盲目修改再試 | `CLAUDE.md` |
| D | 步驟4 validate 必須傳入 sysinfo.json，CHECK 8 GPU 架構為必要項 | `docs/WORKFLOW.md`, `docs/VALIDATION.md` |

---

### 這次作業的特殊情況（供下次參考）

1. **下載機 = 目標機**：正常流程假設有網路的機器和目標機是分開的，但本次目標伺服器本身有網路，直接在目標機上下載。未來若遇到真正離線環境，需另外準備下載機。

2. **Blackwell GPU 最低 open module 版本**：CUDA repo 提供的 `kmod-nvidia-open-dkms` 最低為 610.43.02，無 580.x open module，必須整體升級 driver 到 610.x（CUDA 13.0 向下相容，不影響）。

3. **下載環境限制**：`dnf download --resolve` 在已安裝衝突套件的環境下失敗，需改用 `dnf repoquery --location` 取得 URL 後 `wget` 直接下載。
