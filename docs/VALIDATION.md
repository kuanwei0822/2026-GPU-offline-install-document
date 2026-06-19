# 驗證規範

## 步驟2｜sysinfo.json 驗證規範

### 必要欄位（缺少任何一項 → 修正腳本）
```
system.os_name
system.os_version
system.kernel
system.arch
system.pkg_manager
```

### 格式檢查
```
JSON 格式是否合法
collect_sh_version 是否存在
collected_at 是否有時間戳
```

### 合理性檢查
```
arch 必須是 x86_64 或 aarch64
pkg_manager 必須是 rpm 或 deb
el_version 若是 rpm 系統必須存在（el8 / el9）
kernel 格式是否合理
```

### 警告項目（不阻擋流程，但要記錄）
```
gpu.lspci 是 not available → 警告：無法偵測 GPU 硬體
docker.installed 是 false  → 警告：Docker 未安裝
```

### GPU 架構偵測（影響步驟3下載策略）
從 `gpu.lspci` 字串識別 GPU 架構，決定 kernel module 類型：

```
lspci 含 GB100 / GB102 / GB202（Blackwell）→ 必須用 open-dkms stream
lspci 含 GH100（Hopper）                   → 必須用 open-dkms stream
lspci 含 GA / AD（Ampere / Ada Lovelace）  → 建議用 open-dkms，proprietary 也可
lspci 含 TU（Turing）/ GA 以前             → proprietary 或 open 皆可
```

**識別到 Blackwell/Hopper 時，步驟3下載必須：**
1. `dnf module enable nvidia-driver:open-dkms`（取代 580-dkms 或 latest-dkms）
2. 額外下載 `nvidia-driver-cuda`（含 nvidia-smi，不會被 cuda-drivers 自動拉入）
3. 使用 `kmod-nvidia-open-dkms`（不是 `kmod-nvidia-latest-dkms`）

---

## 步驟4｜靜態驗證腳本

腳本位置：`scripts/validate/`

- `validate_rpm.sh`：RPM 系通用（Rocky / RHEL / AlmaLinux / CentOS）
- `validate_deb.sh`：DEB 系框架（Ubuntu / Debian，待補充）

詳細的 CHECK 邏輯、已知問題、GPU 架構對照表請參考：
- `scripts/validate/notes/known_issues.md`
- `scripts/validate/notes/gpu_arch.md`
- `scripts/validate/notes/whitelist.md`
