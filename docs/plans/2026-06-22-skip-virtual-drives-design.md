# 跳過虛擬磁碟，修正 Google Drive 導致啟動崩潰

日期：2026-06-22
分支：`feat/skip-virtual-drives`
domain spec：`docs/specs/drive-enumeration.md`

## 問題

裝有 Google Drive 桌面版的 Windows 上，Win32 Disk Imager 一啟動就崩潰。

### 崩潰路徑（讀碼確認）

1. `main.cpp:41` 建立 `MainWindow` → 建構子 `mainwindow.cpp:49` 第一件事呼叫 `getLogicalDrives()`。
2. `getLogicalDrives()`（`mainwindow.cpp:1101`）用 `GetLogicalDrives()` 取得所有磁碟機位元遮罩，對「每一台」磁碟機呼叫 `checkDriveType()`。
3. `checkDriveType()`（`disk.cpp:422`）用 `GetDriveType()` 粗分類。只有 `DRIVE_REMOVABLE` 與 `DRIVE_FIXED` 會往下處理（網路碟 `DRIVE_REMOTE`、光碟、RAM 碟落到 `default` 被跳過）。
4. 對通過的磁碟機，`CreateFile` 開啟 `\\.\X:` 後送一連串針對真實區塊儲存裝置的 IOCTL：`GetMediaType()` 的 `IOCTL_DISK_GET_DRIVE_GEOMETRY`、`GetDisksProperty()` 的 `IOCTL_STORAGE_QUERY_PROPERTY` / `IOCTL_STORAGE_GET_DEVICE_NUMBER`、以及 `IOCTL_STORAGE_CHECK_VERIFY2`。

Google Drive 掛載的虛擬磁碟以 `DRIVE_FIXED` 回報，因此通過第 3 步的粗篩進入探測。但它背後是 Google 的虛擬檔案系統驅動、不是真正的磁碟裝置，對它送上述磁碟/儲存 IOCTL 即觸發崩潰。

程式碼除了 `GetDriveType` 的粗分類外，沒有再確認一台 `DRIVE_FIXED` 是否真的對應到實體磁碟——這就是缺少的「跳過虛擬磁碟」判斷。

### 實測佐證（本機已掛 Google Drive）

用 `QueryDosDevice` 列出各磁碟機代號解析到的 NT 裝置目標：

| 代號 | DriveType | FS | NT 目標 |
|------|-----------|----|---------|
| C: D: | Fixed | NTFS | `\Device\HarddiskVolume5` / `HarddiskVolume2` |
| E: F: | Removable | - | `\Device\HarddiskVolume9` / `HarddiskVolume10` |
| W: Y: | **Fixed** | FAT32 | `\Device\Volume{f5ae2bcb-da02-...}` / `{...da03-...}`（Google Drive 虛擬碟） |
| U: V: X: Z: | Network | - | `\Device\LanmanRedirector\...`（已被 DRIVE_REMOTE 擋下） |

關鍵分界：真實磁碟卷（含 USB/SD）一律解析為 `\Device\HarddiskVolume<N>`；Google Drive 虛擬碟解析為 `\Device\Volume{GUID}`，且以 `DRIVE_FIXED` 回報而通過粗篩。

確定性：列舉路徑與缺少虛擬碟判斷＝讀碼確認。「IOCTL 對 Google 驅動觸發崩潰的確切位置」＝本機資料強力佐證的假設，最終由 build + 實機啟動驗收（見驗證策略）。

## 方案決策

採 Option A：在 `checkDriveType` 開頭、開啟裝置與送 IOCTL 之前，加一道純使用者模式（`QueryDosDevice`）的守門，只放行解析到 `\Device\HarddiskVolume...` 的磁碟機。

否決的替代方案：
- Option B（SEH `__try/__except` 包住探測 IOCTL）：治標、仍會去戳 Google 驅動，且 MinGW 對 SEH 支援有限、脆弱。
- Option C（只接受 `DRIVE_REMOVABLE`、完全不處理 `DRIVE_FIXED`）：會漏掉以 `DRIVE_FIXED` 回報的合法 USB 隨身碟（`disk.cpp:443` 註解明講有這種情況），犧牲功能。

## 設計

### 過濾器位置

放進 `checkDriveType()`（`disk.cpp:422`）最前面。此函式是判定「是否為可成像目標」的唯一關卡，同時被以下兩條路徑呼叫，放這裡可一次覆蓋：

- 啟動列舉：`getLogicalDrives()` → `checkDriveType()`（`mainwindow.cpp:1119`）
- 熱插拔：`nativeEvent()` 的 `DBT_DEVICEARRIVAL` → `checkDriveType()`（`mainwindow.cpp:1175`，例如 Google Drive 在程式啟動後才上線時）

兩條路徑傳入的 `name` 都是 `\\.\X:\` 形式，`name[4]` 為磁碟機代號字母。

### 邏輯

1. 從 `name[4]` 組出 `"X:"`（字母 + 冒號，不含斜線與 `\\.\`）。
2. 呼叫 `QueryDosDeviceW(L"X:", buf, n)` 取得 NT 裝置目標。此呼叫不開啟裝置、不送 IOCTL，不碰 Google Drive 驅動，本質上不會觸發崩潰。
3. 以 `QString::fromWCharArray` 包目標字串，`startsWith("\\Device\\HarddiskVolume", Qt::CaseInsensitive)` 為真才繼續跑既有邏輯；否則 `return false` 跳過。
4. `QueryDosDevice` 失敗（回傳 0）時當成跳過——代號來自 `GetLogicalDrives` 必定存在，失敗極罕見，以「絕不崩潰」為優先。

### 範圍與風格

- 不新增設定選項：磁碟映像工具本就不該寫到虛擬碟，永遠跳過即可（YAGNI）。
- 用 Qt 字串 API（`QString` / `Qt::CaseInsensitive`），與 `disk.cpp` 既有風格一致。
- 無 UI 字串變動，不動 `src/lang/*.ts` 翻譯檔。

## 錯誤處理

- `QueryDosDevice` 緩衝區不足（極少見）視為失敗 → 跳過該磁碟機，不崩潰。
- 此守門先於既有的 `CreateFile` / IOCTL，原本對真實磁碟的錯誤對話框行為完全不變。

## 測試 / 驗證策略

本機（已掛 Google Drive W:/Y:）build 後實機啟動：
1. 不再崩潰，且裝置清單只列出實體碟（不含 W:/Y:）。
2. 插入一支 USB 隨身碟，確認熱插拔路徑（`DBT_DEVICEARRIVAL`）仍能正常加入。
3. 確認既有的 SD/USB 讀寫流程不受影響。

## GPL / 文件義務（依 CLAUDE.md checklist）

- `disk.cpp` 標頭於原著作權行下方加 `Copyright (C) 2026 Frank`（本 fork 首次改到此檔）。
- `Changelog.txt` 最上方「Release 1.0.x (fork)」段補一條：修正 Google Drive 等虛擬磁碟導致啟動崩潰，列舉時只接受實體磁碟卷。
- commit 訊息描述改了什麼、為什麼。
- 無版本號變動、無 UI 字串變動。
