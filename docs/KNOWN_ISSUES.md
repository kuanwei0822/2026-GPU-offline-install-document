# 已知問題與解法（持續更新）

---

### [Blackwell GPU] GB202（RTX PRO 6000 Blackwell）只支援 open kernel modules

- **影響 GPU**：PCI ID 10de:2bb5，NVIDIA RTX PRO 6000 Blackwell Server Edition（GB202）
- **現象**：安裝 kmod-nvidia-latest-dkms（proprietary）後，dmesg 報 `NVRM: requires use of the NVIDIA open kernel modules`；/dev/nvidia1 不存在；nvidia-smi 無法使用
- **根因**：Hopper / Blackwell（GB100/GB102/GB202）架構僅支援 open source kernel modules，proprietary 模組無法初始化這些 GPU
- **修正**：下載腳本須啟用 `nvidia-driver:open-dkms` module stream；安裝 `kmod-nvidia-open-dkms` 取代 `kmod-nvidia-latest-dkms`
- **規範**：未來遇到 Hopper / Blackwell GPU，下載腳本一律用 `open-dkms` stream

### [Blackwell GPU] nvidia-driver-cuda 未被原始 cuda-drivers 下載包含，導致 nvidia-smi 缺失

- **現象**：安裝完成後 `nvidia-smi: command not found`；`rpm -ql nvidia-driver | grep smi` 只有 HTML 文件
- **根因**：`nvidia-driver` 套件只含 driver 函式庫；`nvidia-smi` 執行檔在 `nvidia-driver-cuda` 套件，而此套件未被 `dnf download --resolve --alldeps cuda-drivers` 的 580-dkms stream 拉入
- **修正**：補下載 `nvidia-driver-cuda`；或切換到 open-dkms stream 重新下載整個 driver 組

### [install] headless server 上 --nodeps 套件清單（持續更新）

下載機為 Rocky 9.7、目標機為 Rocky 9.6 的場景下，以下套件需使用 `--nodeps` 安裝：

| 套件 | 原因 |
|------|------|
| `dkms` | el9_6 kernel-devel 未提供 `kernel-devel-matched` capability |
| `vulkan-loader` | 依賴 mesa-vulkan-drivers（el9_7），需 libLLVM.so.20.1，裝不了；NVIDIA 自備 Vulkan ICD |
| `cuda-nsight-[0-9]*` | 需要 JRE（Java），headless server 未安裝 |
| `nsight-compute*` | 需要 X11/Qt GUI 函式庫，headless server 未安裝 |
| `nsight-systems*` | 需要 libxcb-icccm / libxcb-image / libxkbcommon-x11，headless server 未安裝 |

### [install] case 白名單完整套件前綴（v8 確認有效）

以下前綴（`name*` 格式）已在 case01-20260527 實際安裝驗證：

**driver 步驟：** `nvidia-*`、`libnvidia-*`、`kmod-nvidia-*`、`egl-wayland*`、`egl-x11*`、`egl-gbm*`、`libglvnd*`、`libvdpau*`、`dkms*`（nodeps）、`vulkan-loader*`（nodeps）

**cuda 步驟：** `cuda-nsight-[0-9]*`（nodeps）、`nsight-compute*`（nodeps）、`nsight-systems*`（nodeps）、`cuda-*`、`libcublas*`、`libcuda*`、`libcufft*`、`libcufile*`、`libcurand*`、`libcusolver*`、`libcusparse*`、`libnpp*`、`libnvfatbin*`、`libnvjitlink*`、`libnvjpeg*`、`libnvptxcompiler*`、`libnvvm*`

**cudnn 步驟：** `libcudnn*`、`cudnn9*`

**container-toolkit 步驟：** `libnvidia-container*`、`nvidia-container*`

### [download_el9] cuda-drivers 版本號格式不被 dnf 接受

- **現象**：`dnf download cuda-drivers-580.159.04` 報 `No package cuda-drivers-580.159.04 available`
- **根因**：dnf 套件名稱不包含版本號；`cuda-drivers-580.159.04-1.el9.x86_64.rpm` 的 dnf 套件名稱是 `cuda-drivers`，版本是獨立欄位
- **修正**：改用 `cuda-drivers`（不帶版本），dnf 自動抓 repo 中最新版本

### [v1→v2] _error 欄位字串未加 JSON 引號

- **觸發版本**：collect_v1.sh
- **現象**：`nvidia_smi_error`（及所有 `_error` 欄位）當值為字串時，直接輸出 shell 變數（如 `command not found`），未加 JSON 雙引號，導致 sysinfo.json 格式無效
- **根因**：heredoc 使用 `${XXX_ERROR}` 直接展開，變數內容為 `null`（合法 JSON）或普通字串（不合法 JSON）
- **修正**：collect_v2.sh 新增 `json_str_or_null()` helper，所有 `_error` 欄位改為 `$(json_str_or_null "${XXX_ERROR}")` 輸出
- **案例**：case01-20260527，sysinfo.json 第22行，已手動修正繼續流程

### [download_supplement] --alldeps 導致系統套件多目錄重複

- **觸發版本**：download_el9_supplement_v1.sh
- **現象**：validate CHECK 5 報大量重複套件，但版本完全相同（bash、glibc、ncurses 等）
- **根因**：`dnf download --resolve --alldeps` 遞迴解析所有間接依賴，四個子目錄各自下載一份系統依賴
- **修正**：補充下載腳本改為 `--resolve`（移除 `--alldeps`），只補缺件本身
- **規範**：補充下載腳本一律不使用 `--alldeps`

### [validate CHECK 5] 同版本多目錄副本誤判為版本衝突

- **觸發版本**：validate_el9.sh（case01-20260527 首次驗證）
- **現象**：CHECK 5 FAIL，報大量「重複套件」，實際上版本完全相同
- **根因**：`uniq -d` 前未做 `sort -u`，同版本出現兩次就被判重複
- **修正**：加入 `sort -u` 先去重，再偵測是否有不同版本

### [validate CHECK 6] 桌面/系統套件誤判為 NVIDIA 依賴缺失

- **觸發版本**：validate_el9.sh（case01-20260527 首次驗證）
- **現象**：CHECK 6 FAIL，報缺失 flatpak、java-17-openjdk、libcanberra、libgtk 等
- **根因**：過濾邏輯為負向比對，列舉不完整，其餘全被誤標 NVIDIA 缺失
- **修正**：改為正向比對——包含 nvidia/cuda/cudnn/nccl/nvtx/cufft 等關鍵字才算 NVIDIA 缺失
- **規範**：未來新增 NVIDIA 子套件（NCCL、TensorRT 等）須將關鍵字補入正向清單

### [install_v1] el9_7 系統套件在 el9_6 目標機引發版本鎖定衝突

- **觸發版本**：install_v1.sh（case01-20260527）
- **現象**：nvidia_driver 步驟 `rpm -Uvh --force` 失敗，報 `audit-libs/systemd/elfutils/libxml2 等版本不符`
- **根因**：下載機為 Rocky Linux 9.7，`--alldeps` 拉入 el9_7 系統套件，但目標機為 Rocky Linux 9.6 且已安裝 el9_6 版本，其他套件對這些 el9_6 版本有精確版號依賴
- **修正**：install_v2.sh 的 `rpm_install_dir` 新增過濾邏輯：非 NVIDIA/CUDA/DKMS 的系統套件，若目標機已安裝則跳過
- **規範**：下載機 OS 版本應盡量與目標機一致；若無法一致，install.sh 必須跳過已安裝系統套件

### [install_v1] DKMS 3.4.1 要求 kernel-devel-matched，el9_6 kernel-devel 未提供此 capability

- **觸發版本**：install_v1.sh（case01-20260527）
- **現象**：`rpm -Uvh dkms` 失敗：`(kernel-devel-matched if kernel-core) is needed by dkms-3.4.1-1.el9.noarch`
- **根因**：DKMS 3.x 的 weak dependency 要求 kernel-devel 提供 `kernel-devel-matched` capability；el9_6 舊版 kernel-devel 未提供
- **修正**：install_v2.sh 對 DKMS 改用 `rpm -Uvh --force --nodeps`
- **規範**：DKMS 安裝一律加 `--nodeps`

### [install_v2] 未安裝的 el9_7 桌面套件（mesa-vulkan-drivers）被選入安裝清單，引發間接依賴衝突

- **觸發版本**：install_v2.sh（case01-20260527）
- **現象**：`mesa-vulkan-drivers-25.0.7-5.el9_7` 被選入安裝，報 `libLLVM.so.20.1 is needed`
- **根因**：v2 邏輯「非白名單 + 未安裝 → 加入清單」，mesa-vulkan-drivers 不在目標機，被選入；但其 el9_7 依賴在 el9_6 無法滿足
- **修正**：install_v3.sh 改為嚴格白名單邏輯
- **規範**：install.sh 套件安裝邏輯應使用正向白名單（只裝 GPU 套件）

### [install_v3] 白名單遺漏 nvidia-driver-libs 的直接前置套件（egl-gbm、libvdpau、vulkan-loader）

- **觸發版本**：install_v3.sh（case01-20260527）
- **現象**：`nvidia-driver-libs` 安裝失敗：`egl-gbm >= 1.1.2`、`libvdpau >= 0.5`、`vulkan-loader` 缺失
- **修正**：install_v4.sh 白名單加入 `egl.gbm|libvdpau|vulkan.loader`

### [install_v4] vulkan-loader 依賴 mesa-vulkan-drivers（el9_7）無法在 el9_6 機器安裝

- **觸發版本**：install_v4.sh（case01-20260527）
- **現象**：`mesa-vulkan-drivers(x86-64) is needed by vulkan-loader-1.4.313.0-1.el9.x86_64`
- **根因**：vulkan-loader 1.4.x RPM requires mesa-vulkan-drivers；mesa-vulkan-drivers el9_7 需要 libLLVM.so.20.1，el9_6 只有 libLLVM.so.19.1
- **修正**：install_v5.sh 將 vulkan-loader 加入 `--nodeps` 安裝群組；NVIDIA 自備 Vulkan ICD
- **規範**：vulkan-loader 一律用 `--nodeps` 安裝

### [install_v5] case 模式結尾加 '-*' 導致裸名套件全部漏匹配

- **觸發版本**：install_v5.sh（case01-20260527）
- **現象**：「安裝：12 個 RPM（--nodeps 0）」，dkms/vulkan-loader/egl-wayland 等全部消失
- **根因**：rpm `%{NAME}` 返回純套件名（如 `dkms`），v5 的 case 寫 `dkms-*` 要求後面有 `-`，純名稱不匹配
- **修正**：install_v6.sh 改用 `dkms*`、`egl-wayland*` 等（glob 不加 `-`）
- **規範**：rpm case 白名單模式一律用 `pkgname*` 而非 `pkgname-*`

### [install_v6] 白名單遺漏 CUDA 核心函式庫 libcublas、libnvjitlink；cuda-nsight 需要 jre

- **觸發版本**：install_v6.sh（case01-20260527）
- **現象**：CUDA 步驟失敗：`libcublas-13-0 is needed`、`libnvjitlink-13-0 is needed`、`jre >= 1.7 is needed by cuda-nsight-13-0`
- **修正**：install_v7.sh 加入 `libcublas*`、`libnvjitlink*`；`cuda-nsight-[0-9]*` 移至 --nodeps 群組
- **規範**：server 環境 cuda-nsight-* 一律 --nodeps；libcublas、libnvjitlink 是 ML 框架必需庫

### [install_v7] nsight-systems / nsight-compute 需要 X11 xcb GUI 函式庫，headless server 未安裝

- **觸發版本**：install_v7.sh（case01-20260527）
- **現象**：`libxcb-icccm.so.4 / libxcb-image.so.0 / libxkbcommon-x11.so.0 is needed by nsight-systems`
- **修正**：install_v8.sh 將 `nsight-compute*` 和 `nsight-systems*` 移至 `--nodeps` 群組
- **規範**：nsight-compute / nsight-systems 在 headless server 一律 --nodeps

### [Windows] Git Bash 執行 Docker 路徑被轉換

- **觸發環境**：Windows + Git Bash（Bash tool）執行 `docker run -v /work:...`
- **現象**：`/work` 被 Git Bash 自動轉為 `C:/Program Files/Git/work`，容器掛載失敗
- **根因**：Git Bash（MSYS/MinGW）的 POSIX 路徑轉換行為
- **修正**：Windows 上執行 Docker 命令一律改用 PowerShell
