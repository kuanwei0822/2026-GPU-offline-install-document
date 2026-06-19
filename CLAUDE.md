# GPU 離線安裝自動化工作手冊

## 專案背景

本專案目的是協助在**完全離線的 Linux 環境**中，正確安裝 GPU 相關套件。
每次作業的核心是：在有網路的機器上準備好所有套件，再帶入離線目標機安裝。

### 目標套件（依安裝順序）

```
1. Kernel 相關套件（kernel-devel、kernel-headers、編譯工具）
2. NVIDIA Driver
3. CUDA Toolkit（版本須與 NVIDIA Driver 相容，以 NVIDIA 官方相容表為準）
4. cuDNN（版本須與 CUDA Toolkit 相容，以 NVIDIA 官方相容表為準）
5. NVIDIA Container Toolkit（版本須與 NVIDIA Driver 相容，以官方文件為準）
```

### 版本相依關係

每次作業前須查詢官方文件確認當前相容版本，不使用寫死的版本號。

```
NVIDIA Driver
  ↓ 強相依（查詢：https://docs.nvidia.com/cuda/cuda-toolkit-release-notes）
CUDA Toolkit
  ↓ 強相依（查詢：https://docs.nvidia.com/deeplearning/cudnn/support-matrix）
cuDNN

NVIDIA Driver
  ↓ 強相依（查詢：https://docs.nvidia.com/datacenter/cloud-native/container-toolkit）
Container Toolkit
```

### 支援的 Linux 發行版

- RHEL 系列：Rocky Linux、AlmaLinux、CentOS、RHEL
- Debian 系列：Ubuntu、Debian
- 其他：SUSE、Oracle Linux

---

## Claude 行為規範

### 遇到失敗時：先確認狀態，不盲目修改再試

**背景**：case01-20260527 中，download_open_module.sh 多次失敗，Claude 每次都修改腳本繼續嘗試，而非停下來確認。執行人需要主動制止才改為確認模式。

**規範**：
- 任何指令或腳本第一次失敗後，**先請執行人確認當前狀態**，再決定下一步
- 不能在未確認狀態的情況下，連續修改腳本或指令超過一次
- 確認的形式：請執行人執行診斷指令，回覆輸出後再判斷
- 若兩次嘗試後仍失敗，主動建議進入 **情況D（手動排查）**

### 驗證失敗時：主動提出診斷指令

- 安裝或驗證步驟失敗後，Claude 應**主動提供診斷指令**，而不等執行人要求
- 診斷指令應一次完整列出，讓執行人一次執行完，不來回多輪

---

## 文件索引

| 文件 | 內容 |
|------|------|
| [docs/WORKFLOW.md](./docs/WORKFLOW.md) | 工作流程（步驟1～7）、核心設計原則、角色定義 |
| [docs/SCRIPTS.md](./docs/SCRIPTS.md) | collect.sh 設計規範、sysinfo.json 標準格式 |
| [docs/VALIDATION.md](./docs/VALIDATION.md) | sysinfo 驗證規範、GPU 架構偵測、靜態驗證腳本說明 |
| [docs/INSTALL_SPEC.md](./docs/INSTALL_SPEC.md) | install.sh 設計規範（設計哲學、觀測原則、白名單、log 規則）|
| [docs/KNOWN_ISSUES.md](./docs/KNOWN_ISSUES.md) | 已知問題與解法（持續更新）|
| [docs/RETROSPECTIVE.md](./docs/RETROSPECTIVE.md) | 作業檢討紀錄（實際過程、根因分析、改善結論）|

---

## 腳本庫位置

| 腳本類型 | 位置 | 說明 |
|---------|------|------|
| collect 腳本 | `scripts/collect/` | 新案子起點，取最新版本 |
| install 腳本 | `scripts/install/` | 新案子起點，取最新版本 |
| validate 腳本 | `scripts/validate/` | RPM/DEB 通用靜態驗證 |
| download 腳本 | `scripts/download/` | 各類下載腳本 |

---

## 資料夾規則

- `scripts/` 只增不刪，永久保留所有版本
- 新案子開始時，從 `scripts/` 複製最新版到 `projects/{案子}/`
- `projects/{案子}/` 內的腳本是該次快照，不受後續腳本庫更新影響
- 案子命名：`客戶名_YYYYMMDD`，例如 `companyA_20260527`
- `docs/` 與 `scripts/` 有異動時，同步更新 `docs/KNOWN_ISSUES.md`

---

## 靜態驗證知識庫

`scripts/validate/notes/` 包含跨案子累積的知識，**開新案子前必須查閱**：

| 文件 | 用途 |
|------|------|
| `scripts/validate/notes/gpu_arch.md` | GPU 架構對照表，決定 open/proprietary module stream |
| `scripts/validate/notes/whitelist.md` | 套件白名單前綴，install.sh 白名單的知識來源 |
| `scripts/validate/notes/known_issues.md` | 驗證步驟已知問題 |
