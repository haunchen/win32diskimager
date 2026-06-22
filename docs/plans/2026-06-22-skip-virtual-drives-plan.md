# 跳過虛擬磁碟（修正 Google Drive 啟動崩潰）Implementation Plan

Goal: 在磁碟列舉時跳過非實體磁碟（Google Drive 虛擬碟、網路碟、SUBST、RAM 碟），讓裝有 Google Drive 的 Windows 啟動 Win32 Disk Imager 不再崩潰。

Architecture: 在 `checkDriveType()`（判定「是否為可成像目標」的唯一關卡，啟動列舉與熱插拔都會經過）最前面加一道純使用者模式守門：用 `QueryDosDevice` 把磁碟機代號解析成 NT 裝置目標，只放行 `\Device\HarddiskVolume...` 開頭的真實磁碟卷。此呼叫不開啟裝置、不送 IOCTL，從源頭避免觸碰虛擬碟驅動而崩潰。

Tech Stack: C++ / Qt 5.7 / MinGW 5.3 / qmake；Win32 API `QueryDosDeviceW`。

Spec: `docs/specs/drive-enumeration.md`

Design: `docs/plans/2026-06-22-skip-virtual-drives-design.md`

測試前提：本專案無自動化單元測試框架，且守門邏輯依賴 Win32 API 與實際掛載的磁碟。因此驗證採「編譯通過 + 在已掛 Google Drive 的本機實機啟動」，這正是真實重現環境。

---

### Task 1: 加入 isPhysicalDiskVolume 守門

Implements: `drive-enumeration.md` #R1, #R2, #R3, #R4

Files:
- Modify: `src/disk.cpp`（新增 static helper + `checkDriveType` 開頭守門 + 檔案標頭著作權行）

Step 1: 在 `src/disk.cpp` 的 `bool checkDriveType(char *name, ULONG *pid)` 函式定義「之前」插入 helper。

用 Edit，old_string：

```cpp
bool checkDriveType(char *name, ULONG *pid)
{
    HANDLE hDevice;
```

new_string：

```cpp
// Return true only if drive letter 'driveLetter' (e.g. 'G') maps to a real
// physical-disk volume.  Virtual filesystems (Google Drive for Desktop),
// network redirectors, SUBST and RAM disks resolve to other NT device targets
// and must be skipped: probing them below with disk/storage IOCTLs can crash
// the app.  QueryDosDevice is a pure user-mode lookup that neither opens the
// device nor issues IOCTLs, so it is safe even against such drivers.
static bool isPhysicalDiskVolume(char driveLetter)
{
    wchar_t deviceName[3] = { (wchar_t)driveLetter, L':', L'\0' };
    wchar_t targetPath[MAX_PATH];
    DWORD len = QueryDosDeviceW(deviceName, targetPath, MAX_PATH);
    if (len == 0)
    {
        // The letter came from GetLogicalDrives so it exists; a failure here
        // is rare (e.g. buffer too small for a long network target).  Prefer
        // "never crash" over "never miss" and treat it as non-physical.
        return false;
    }
    QString target = QString::fromWCharArray(targetPath);
    return target.startsWith(QLatin1String("\\Device\\HarddiskVolume"), Qt::CaseInsensitive);
}

bool checkDriveType(char *name, ULONG *pid)
{
    HANDLE hDevice;
```

Step 2: 在 `checkDriveType` 開頭、`slashify` 呼叫之前插入守門。

用 Edit，old_string：

```cpp
    DWORD cbBytesReturned;

    // some calls require no tailing slash, some require a trailing slash...
```

new_string：

```cpp
    DWORD cbBytesReturned;

    // Skip any drive that is not backed by a real physical disk (Google Drive
    // virtual drive, network share, SUBST, RAM disk).  name is "\\.\X:\", so
    // name[4] is the drive letter.  Doing this before opening a handle avoids
    // sending storage IOCTLs to a virtual filesystem driver, which crashes.
    if ( !isPhysicalDiskVolume(name[4]) )
    {
        return(retVal);
    }

    // some calls require no tailing slash, some require a trailing slash...
```

註：此處 `retVal` 已宣告為 `false`，且守門在 `slashify` 配置記憶體之前，提早 return 無記憶體洩漏。

Step 3: 在 `src/disk.cpp` 檔案標頭加本 fork 著作權行（CLAUDE.md GPL checklist，本檔首次被改）。

用 Edit，old_string：

```
 *                 https://sourceforge.net/projects/win32diskimager/  *
 **********************************************************************/
```

new_string：

```
 *                 https://sourceforge.net/projects/win32diskimager/  *
 *  Copyright (C) 2026 Frank                                          *
 **********************************************************************/
```

（尾端 `*` 對齊為純美觀；若空白數略有出入不影響編譯。）

Step 4: 編譯確認通過
Run: `cd src && qmake && mingw32-make`（或在 repo 根跑 `compile.bat`）
Expected: 編譯無錯誤，產出 Release 執行檔。若 Qt/MinGW 不在當前 shell 的 PATH，改由使用者執行 `compile.bat`。

Step 5: Commit
Run: `git add src/disk.cpp && git commit -m "fix: 列舉時跳過虛擬磁碟，修正 Google Drive 導致啟動崩潰"`

---

### Task 2: 更新 Changelog

Implements: （文件義務，無 spec requirement）

Files:
- Modify: `Changelog.txt`

Step 1: 把 Release 1.0.1 段的 TODO 佔位行換成本次修正。

用 Edit，old_string：

```
Changes in this fork:
- (TODO: list bug fixes here as they are made)
```

new_string：

```
Changes in this fork:
- 修正裝有 Google Drive（或其他以固定磁碟身分回報的虛擬磁碟）時，
  程式一啟動即崩潰的問題。磁碟列舉改為先以 QueryDosDevice 確認磁碟機
  對應到實體磁碟卷（\Device\HarddiskVolume），非實體者（虛擬碟、網路碟、
  SUBST、RAM 碟）一律跳過，不再對其送出磁碟／儲存 IOCTL。
```

Step 2: Commit
Run: `git add Changelog.txt && git commit -m "docs: Changelog 補上跳過虛擬磁碟的修正說明"`

---

### Task 3: 實機驗證（本機已掛 Google Drive W:/Y:）

Implements: 驗收 `drive-enumeration.md` S1, S2, S3

Files:（無，純驗證）

Step 1: 以系統管理員身分啟動編譯好的 Win32DiskImager.exe。
Expected: 程式正常開啟、不崩潰（驗 S1 / #R2）。

Step 2: 檢查裝置下拉清單。
Expected: 不含 Google Drive 的 W:/Y:，也不含網路碟 U/V/X/Z（驗 S1 / S3 / #R1）。既有可成像的可移除／USB 磁碟仍正常出現。

Step 3: 插入一支 USB 隨身碟。
Expected: 該磁碟機被加入下拉清單（驗 S2 / #R3 / #R4，確認熱插拔 `DBT_DEVICEARRIVAL` 路徑未被破壞）。

Step 4: （回歸）對一張 SD/USB 做一次讀或寫的小範圍操作，確認既有讀寫流程不受影響。

若任一步未過，退回 Task 1 檢視守門條件與 `name[4]` 取字母是否正確。
