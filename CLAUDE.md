# CLAUDE.md

給在這個 repo 工作的 AI 助手（與開發者）的專案指引。

## 專案簡介

這是 Win32 Disk Imager 的非官方 fork，維護者 Frank（GitHub: @haunchen）。
原專案是把 raw image（`.img`）讀寫到 SD / USB 裝置的 Windows 工具，需要系統管理員權限執行。

- 語言/框架：C++ 搭配 Qt 5.7 + MinGW 5.3，qmake 建置。
- 上游版本：1.0.0（SourceForge）。本 fork 從 SourceForge master 的未修改原始碼開始（見第一個 commit）。
- 原作者：Justin Davis <tuxdavis@gmail.com>；上游由 ImageWriter developers 維護。

## 授權與 GPL 義務（最重要，動任何程式碼前先讀）

本專案授權為 GPL-2.0-or-later（原始檔標頭寫明「version 2, or at your option any
later version」）。這不是建議，是法律義務：

- 不能更換授權。整個專案必須維持 GPL，不可改成 MIT / Apache 等寬鬆授權，也不可
  把任何檔案重新授權。
- 保留所有原始著作權標頭與授權文字。每個原始檔開頭的 GPL 標頭、Justin Davis 與
  ImageWriter developers 的著作權行、SourceForge 連結都必須原樣保留，不可刪除或改寫。
- 標註你的修改（GPL §2a）。修改任何既有檔案時，要讓變更可追溯：
  - 每個 bug 修正用獨立、訊息清楚的 commit。
  - 在 `Changelog.txt` 最上方的 fork 版本段落記錄改了什麼。
  - 對既有檔案做實質修改時，可在該檔標頭原著作權行下方加一行
    `Copyright (C) 2026 Frank`，不要動到上游既有的著作權行。
- 新增的檔案：沿用相同的 GPL 標頭，著作權行寫
  `Copyright (C) 2026 Frank`。
- 發布二進位檔的義務：若提供編譯好的 `.exe`，必須一併提供對應的完整原始碼
  （放在這個 GitHub repo 即滿足）。
- 第三方元件 Qt 是 LGPL-2.1。原始碼層面沒問題；但若 Release 附帶 Qt DLL，必須遵守
  LGPL——附上 LGPL 文字、確保使用者能替換或重新連結 Qt，不要把 Qt 靜態連進執行檔。
  MinGW runtime 依其各自授權。

簡言之：可以自由修改與再發布，但要保留授權與所有原始聲明、清楚標註自己的修改、
整體維持 GPL。

## 程式碼地圖

- `src/main.cpp` — 進入點，載入翻譯、建立主視窗。
- `src/mainwindow.{h,cpp,ui}` — GUI 與主要邏輯（讀/寫/驗證、checksum、設定）。
- `src/disk.{h,cpp}` — Win32 裝置層：IOCTL、取得 handle、讀寫磁區、鎖定/卸載磁碟區。
- `src/droppablelineedit.{h,cpp}` — 支援拖放的輸入框。
- `src/elapsedtimer.{h,cpp}` — 進度/耗時計算。
- `src/lang/diskimager_*.ts` — Qt Linguist 翻譯檔（含 zh_TW、zh_CN）。
- `src/DiskImager.pro` — qmake 專案檔；`DiskImager.rc` / `DiskImager.manifest` 處理
  版本資源與「要求系統管理員權限」的 manifest。
- `setup.iss` — Inno Setup 安裝檔腳本。
- `Win32SysInfo/` — 輔助的系統資訊小工具（與主程式分離）。

## 建置

```bat
compile.bat   :: 等同 cd src && qmake && mingw32-make
clean.bat     :: 清理 build 產物
```

或用 Qt Creator 開 `src/DiskImager.pro`。程式需要系統管理員權限執行，除錯時請以
管理員身分啟動 Qt Creator。

## 開發慣例與注意事項

- Commit：每個修正獨立 commit，訊息說清楚改了什麼與為什麼（同時滿足 GPL 修改標註）。
  第一個 commit 是未修改的上游基準，方便 diff 出本 fork 的改動。
- 換行：repo 內檔案原始為 LF，git blob 保留 LF。`.bat` 檔在 Windows 需要 CRLF，
  不要對它們做 LF 正規化。
- 含中文/CJK 的檔案（特別是 `src/lang/*.ts`）一律用 Edit / Write 工具修改；
  在 Windows 上避免用未指定 encoding 的 PowerShell `Get-Content -Raw` 做替換，會
  產生 mojibake。
- 修改字串後要更新翻譯：新字串需同步進 `.ts` 檔（Qt Linguist）並 lrelease 成 `.qm`。
- 版本號散落在多處，改版本時一起更新：`src/DiskImager.pro`（VERSION）、
  `src/DiskImager.rc`（FILEVERSION / PRODUCTVERSION / 字串）、`Changelog.txt`、
  `README.md`。
- `*.pro.user`、`build*/`、`Release/`、`Debug/`、`*.exe`、`*.dll`、`*.qm` 已在
  `.gitignore` 內，不要 commit。
