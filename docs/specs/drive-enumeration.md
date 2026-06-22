---
domain: drive-enumeration
status: active
created: 2026-06-22
last_modified: 2026-06-22
---

# Drive Enumeration

決定哪些磁碟機會被當成可讀寫映像的目標，並列入裝置清單的行為。

## Requirements

### R1: 只列出實體磁碟卷
- **Level**: MUST
- **Description**: 列舉目標磁碟機時，只接受對應到實體區塊儲存裝置（USB、SD、內接硬碟等）的磁碟機。對應到虛擬檔案系統的磁碟機（如 Google Drive 桌面版掛載的磁碟）、網路重導磁碟、SUBST 與 RAM 碟，皆不得列入裝置清單。

### R2: 探測非實體磁碟機不得導致崩潰
- **Level**: MUST
- **Description**: 系統存在以「固定磁碟（fixed）」身分回報的虛擬磁碟（如 Google Drive）時，程式啟動與裝置插拔過程都不得崩潰。判定一台磁碟機是否為實體磁碟，不得對非實體磁碟機送出針對真實磁碟設計的儲存／磁碟 IOCTL。

### R3: 保留合法可移除與 USB 固定磁碟
- **Level**: MUST
- **Description**: 以「可移除（removable）」身分回報的磁碟，以及以「固定」身分回報但實際為 USB／SD／MMC 的磁碟，仍須正常列入裝置清單，行為與既有版本一致。

### R4: 啟動列舉與熱插拔行為一致
- **Level**: MUST
- **Description**: 程式啟動時的磁碟列舉，與執行中收到裝置抵達通知時的處理，須套用相同的「是否為合法目標」判定，不得只在其中一條路徑過濾。

## Scenarios

### S1: Google Drive 已掛載時啟動
- **Given**: Windows 已掛載 Google Drive 桌面版的虛擬磁碟（以 fixed 身分回報，NT 目標非 `\Device\HarddiskVolume`）
- **When**: 啟動 Win32 Disk Imager
- **Then**: 程式正常開啟、不崩潰，裝置清單不含該虛擬磁碟
- **Implements**: #R1, #R2

### S2: 插入 USB 隨身碟
- **Given**: 程式執行中
- **When**: 插入一支 USB 隨身碟（不論回報為 removable 或 USB-bus 的 fixed）
- **Then**: 該磁碟機被加入裝置清單
- **Implements**: #R3, #R4

### S3: 網路磁碟與 SUBST
- **Given**: 系統存在網路磁碟機或 SUBST 磁碟機
- **When**: 啟動或重新列舉裝置
- **Then**: 這些磁碟機不出現在裝置清單
- **Implements**: #R1

## Design Decisions

### D1: 以 QueryDosDevice 的 NT 目標分辨實體磁碟
- **Decision**: 在判定磁碟機類型、開啟裝置 handle 之前，先用 `QueryDosDevice` 取得磁碟機代號的 NT 裝置目標，僅放行目標以 `\Device\HarddiskVolume` 開頭者。
- **Rationale**: 本機實測顯示真實磁碟卷（含 USB/SD）一律解析為 `\Device\HarddiskVolume<N>`，而 Google Drive 虛擬碟解析為 `\Device\Volume{GUID}` 且以 fixed 回報、會通過既有的 `GetDriveType` 粗篩。`QueryDosDevice` 是純使用者模式呼叫，不開裝置、不送 IOCTL，從源頭避免觸碰虛擬磁碟驅動而崩潰。
- **Date**: 2026-06-22

### D2: 不提供「顯示虛擬磁碟」設定選項
- **Decision**: 永遠跳過非實體磁碟，不加開關。
- **Rationale**: 磁碟映像工具不存在寫入虛擬碟的合理需求；多一個選項只增加維護面與誤用風險（YAGNI）。
- **Date**: 2026-06-22

### D3: QueryDosDevice 失敗時保守跳過
- **Decision**: `QueryDosDevice` 回傳失敗時，視為非實體磁碟而跳過。
- **Rationale**: 磁碟機代號來自 `GetLogicalDrives`、必定存在，失敗極罕見；以「絕不崩潰」優先於「絕不漏列」。
- **Date**: 2026-06-22
