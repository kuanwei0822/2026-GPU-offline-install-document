# GPU 離線安裝常見問題與解法

整理自 case01-20260527（Rocky Linux 9.6 × NVIDIA RTX PRO 6000 Blackwell × CUDA 13.0）

---

## 一、資訊收集（collect.sh）

### 1. `_error` 欄位值未加 JSON 引號
- **現象**：`sysinfo.json` 格式無效，`_error` 欄位的字串值沒有加雙引號
- **根因**：heredoc 直接展開 shell 變數，`null` 合法但普通字串不合法
- **解法**：新增 `json_str_or_null()` helper，所有 `_error` 欄位改用此函數輸出

---

## 二、下載（download.sh）

### 2. `cuda-drivers-版本號` 格式被 dnf 拒絕
- **現象**：`dnf download cuda-drivers-580.159.04` 報 `No package available`
- **根因**：dnf 套件名稱不含版本號，版本是獨立欄位
- **解法**：改用 `cuda-drivers`（不帶版本），dnf 自動抓最新

### 3. `--alldeps` 導致多目錄重複套件
- **現象**：四個子目錄（driver/、cuda/、cudnn/、container-toolkit/）各自下載一份系統依賴
- **根因**：`dnf download --resolve --alldeps` 遞迴解析所有間接依賴
- **解法**：補充下載腳本改用 `--resolve`（移除 `--alldeps`）

### 4. Blackwell GPU 需要 open-dkms stream，且 `nvidia-driver-cuda` 不會被自動拉入
- **現象**：安裝後 `nvidia-smi: command not found`；driver 用 proprietary module 無法驅動 GPU
- **根因**：
  - Blackwell（GB202）只支援 open source kernel modules
  - `nvidia-driver-cuda`（含 nvidia-smi）不在 `cuda-drivers` 的依賴鏈中
  - 下載腳本使用 `nvidia-driver:580-dkms`（proprietary）而非 `open-dkms`
- **解法**：
  - lspci 字串含 GB202/Blackwell/Hopper → 下載時改用 `nvidia-driver:open-dkms` module stream
  - 額外下載 `nvidia-driver-cuda`
  - 使用 `kmod-nvidia-open-dkms`（不是 `kmod-nvidia-latest-dkms`）

**GPU 架構與 module stream 對應：**
| GPU 架構 | PCI 型號 | 需要 |
|----------|---------|------|
| Blackwell | GB100/GB102/GB202 | open-dkms（強制） |
| Hopper | GH100 | open-dkms（強制） |
| Ada Lovelace / Ampere | AD/GA | open-dkms（建議） |
| Turing 以前 | TU | proprietary 或 open 皆可 |

---

## 三、靜態驗證（validate.sh）

### 5. 同版本多目錄副本被誤判為版本衝突
- **現象**：CHECK 5 FAIL，報大量重複套件，實際版本完全相同
- **根因**：`uniq -d` 前未做 `sort -u`，同版本出現兩次就被判重複
- **解法**：加入 `sort -u` 先去重，再偵測是否有不同版本

### 6. 桌面/系統套件被誤判為 NVIDIA 依賴缺失
- **現象**：CHECK 6 FAIL，報缺失 flatpak、java-17-openjdk、libcanberra 等
- **根因**：過濾邏輯為負向比對，列舉不完整
- **解法**：改為正向比對——只有含 nvidia/cuda/cudnn/nccl/nvtx/cufft 等關鍵字才算 NVIDIA 缺失

---

## 四、安裝腳本（install.sh）通用問題

### 7. 下載機 OS 版本（el9_7）vs 目標機 OS 版本（el9_6）衝突
- **現象**：`rpm -Uvh --force` 失敗，報 `audit-libs/systemd/elfutils 版本不符`
- **根因**：`--alldeps` 拉入 el9_7 系統套件，目標機 el9_6 已安裝且有其他套件鎖定精確版號
- **解法**：install.sh 使用白名單邏輯，只裝 GPU 相關套件，系統套件一律跳過

### 8. DKMS 3.4.1 要求 `kernel-devel-matched` capability
- **現象**：`rpm -Uvh dkms` 失敗：`(kernel-devel-matched if kernel-core) is needed`
- **根因**：el9_6 的舊版 kernel-devel 未提供此 capability，但 kernel-devel 實際可用
- **解法**：DKMS 安裝一律加 `--nodeps`

### 9. 未安裝的 el9_7 桌面套件被選入安裝清單，引發間接依賴衝突
- **現象**：`mesa-vulkan-drivers` 被選入，安裝失敗（需要 libLLVM.so.20.1）
- **根因**：「非白名單 + 未安裝 → 裝」的邏輯，桌面套件不在目標機上，但其 el9_7 依賴也無法滿足
- **解法**：改為嚴格白名單（只裝 GPU 套件），其他全跳過

### 10. `rpm %{NAME}` 返回裸名，case 模式加 `-*` 導致漏匹配
- **現象**：`dkms`、`egl-wayland`、`libvdpau` 等套件全部漏選（0 個 --nodeps 套件）
- **根因**：`dkms-3.4.1.rpm` 的 `%{NAME}` 是 `dkms`，模式 `dkms-*` 要求後面有 `-`，不匹配
- **解法**：一律用 `dkms*`、`egl-wayland*`（不加 `-`）

### 11. `vulkan-loader` 依賴 `mesa-vulkan-drivers`（el9_7）
- **現象**：`mesa-vulkan-drivers(x86-64) is needed by vulkan-loader`
- **根因**：vulkan-loader RPM 的 requires 包含 mesa-vulkan-drivers，但 mesa-vulkan-drivers el9_7 需要 libLLVM.so.20.1
- **解法**：vulkan-loader 加入 `--nodeps` 安裝群組（NVIDIA 自備 Vulkan ICD）

### 12. CUDA nsight 套件需要 Java / X11 GUI 函式庫
- **現象**：`jre >= 1.7 is needed by cuda-nsight`；`libxcb-icccm is needed by nsight-systems`
- **根因**：nsight 系列為 GUI profiling 工具，headless server 沒有 Java 和 X11
- **解法**：`cuda-nsight-[0-9]*`、`nsight-compute*`、`nsight-systems*` 全部加入 `--nodeps` 群組

---

## 五、Blackwell GPU 特殊補裝流程

### 13. `kmod-nvidia-open-dkms` 套件名稱與 `kmod-nvidia-latest-dkms` 不同
- **現象**：`dnf download kmod-nvidia-open-latest-dkms` 報 `No package available`
- **根因**：open module 套件名為 `kmod-nvidia-open-dkms`（無 latest 字樣）
- **解法**：使用正確套件名 `kmod-nvidia-open-dkms`

### 14. `dkms remove --all` 刪除 DKMS source，導致無法重建
- **現象**：`dkms autoinstall` 執行後 `dkms status` 空白，kernel modules 消失
- **根因**：`dkms remove MODULE/VERSION --all` 會同時刪除 `/usr/src/nvidia-VERSION/` 的 source code
- **解法**：不要對 DKMS 已安裝的模組呼叫 `remove --all`；重建方式是直接重裝 RPM，讓 post-install script 重新執行

### 15. Blackwell open module 最低版本為 610.43.02，無 580.x open module
- **現象**：只有 `kmod-nvidia-open-dkms-610.43.02`，沒有 `580.x` 版本
- **根因**：NVIDIA 的 open module 套件跳過了某些 minor release
- **解法**：整體 driver 升級至 610.43.02（CUDA 13.0 向下相容，不需重裝 CUDA）

### 16. 610.x 新增 `nvidia-driver-common` 套件，提供 `libnvidia-gpucomp`
- **現象**：rpm --test 報 `libnvidia-gpucomp.so.610.43.02 is needed`，但 libnvidia-gpucomp 沒有獨立 URL
- **根因**：610.x 將 libnvidia-gpucomp 移入 `nvidia-driver-common` 套件
- **解法**：額外下載 `nvidia-driver-common-610.43.02`

### 17. 610.x 新增 `egl-wayland2` 依賴（與原 `egl-wayland` 不同）
- **現象**：rpm --test 報 `egl-wayland2(x86-64) >= 1.0.0 is needed by nvidia-driver-libs`
- **根因**：610.x driver 從 egl-wayland 換為 egl-wayland2
- **解法**：下載 `egl-wayland2`（從 NVIDIA CUDA repo）

---

## 六、工具與環境

### 18. Windows Git Bash 執行 Docker 路徑被轉換
- **現象**：`docker run -v /work:...` 中的 `/work` 被轉為 `C:/Program Files/Git/work`
- **解法**：Windows 上執行 Docker 命令一律改用 PowerShell

### 19. `grep -E` 複雜 alternation 模式在部分環境不相容
- **現象**：`grep -E "^(pattern1|pattern2|...)"` 報 `Unmatched ( or \(`，所有套件被判為不符合
- **根因**：heredoc 傳遞或環境差異導致 grep 無法解析複雜模式
- **解法**：install.sh 中一律改用 bash `case` 語句（glob 模式）取代 `grep -E`

### 20. 終端機截行導致 wget URL 破損
- **現象**：長 URL 被終端機自動換行，wget 收到截斷的 hostname，報 `Invalid host name`
- **解法**：
  - URL 加引號：`wget "https://..."`
  - 或改用 `dnf download` 直接下載（避免手動輸入 URL）
  - 或用 `dnf repoquery --location` 取得 URL 後存檔，再用 `wget -i`

---

## 七、install.sh 白名單套件前綴（v8 驗證有效）

### --nodeps 群組（所有環境）
| 套件前綴 | 原因 |
|---------|------|
| `dkms*` | kernel-devel-matched capability 問題 |
| `vulkan-loader*` | 依賴 mesa-vulkan-drivers（el9_7）|
| `cuda-nsight-[0-9]*` | 需要 JRE（Java）|
| `nsight-compute*` | 需要 X11/Qt GUI 函式庫 |
| `nsight-systems*` | 需要 libxcb-icccm / libxkbcommon-x11 |

### 白名單（正常安裝）
**driver 步驟：**
`nvidia-*`、`libnvidia-*`、`kmod-nvidia-*`、`egl-wayland*`、`egl-x11*`、`egl-gbm*`、`libglvnd*`、`libvdpau*`

**cuda 步驟：**
`cuda-*`、`libcublas*`、`libcuda*`、`libcufft*`、`libcufile*`、`libcurand*`、`libcusolver*`、`libcusparse*`、`libnpp*`、`libnvfatbin*`、`libnvjitlink*`、`libnvjpeg*`、`libnvptxcompiler*`、`libnvvm*`

**cudnn 步驟：**
`libcudnn*`、`cudnn9*`

**container-toolkit 步驟：**
`libnvidia-container*`、`nvidia-container*`
