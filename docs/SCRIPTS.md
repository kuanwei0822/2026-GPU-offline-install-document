# collect.sh 設計規範

## 基本原則
- 必須使用 `root` 權限執行
- 【禁止】任何破壞性指令（rm、install、修改系統設定等），只允許 Query 性質的操作
- 任何指令失敗：記錄錯誤訊息到對應的 `_error` 欄位，繼續執行，不中斷
- 最終一定要產生 `sysinfo.json`，不管過程中有多少錯誤
- 需相容所有常見 Linux 發行版（偵測 rpm/deb 環境）
- 每個指令執行前先用 `command -v` 確認存在

## 容錯規則
```
指令不存在  → 值填 "not available"，_error 填 "command not found"
指令失敗    → 值填 "not available"，_error 填實際錯誤訊息
欄位有值    → _error 填 null
```

## 維護原則
- 資訊只增不減，不刪除已有的收集項目
- 修正後不需要馬上驗證，不阻擋當下補救作業
- 🤖 每次修正原因記錄進 `docs/KNOWN_ISSUES.md`
- 👤 下次進客戶環境順帶帶入新版 `collect.sh`

## 版本命名規則
```
初始版本：collect_v1.sh
第一次修正：collect_v2.sh
第二次修正：collect_v3.sh
...以此類推
```
腳本庫位置：`scripts/collect/`

---

## sysinfo.json 標準格式

```json
{
  "collected_at": "2026-01-01 10:00:00",
  "collect_sh_version": "v1",
  "system": {
    "os_name": "Rocky Linux",
    "os_version": "9.3",
    "el_version": "el9",
    "kernel": "5.14.0-362.8.1.el9_3.x86_64",
    "arch": "x86_64",
    "pkg_manager": "rpm"
  },
  "gpu": {
    "lspci": "NVIDIA Corporation ...",
    "lspci_error": null,
    "nvidia_smi_driver_version": "535.xx.xx",
    "nvidia_smi_error": null
  },
  "existing_packages": {
    "nvidia_driver": "not installed",
    "cuda": "not installed",
    "cudnn": "not installed",
    "container_toolkit": "not installed"
  },
  "compiler": {
    "gcc": "gcc 11.3.1",
    "gcc_error": null,
    "make": "not available",
    "make_error": "command not found",
    "kernel_devel": "not installed",
    "kernel_headers": "not installed"
  },
  "docker": {
    "installed": true,
    "version": "24.0.5",
    "version_error": null
  }
}
```
