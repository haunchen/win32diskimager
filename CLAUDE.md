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
- 標註你的修改（GPL §2a）。所有改動都要可追溯——具體要更新哪些檔案、怎麼填，
  見下方「改動後要維護的文件（checklist）」。
- 發布二進位檔要附對應的完整原始碼（push 到本 repo 即滿足）；若打包 Qt DLL，需另
  遵守 LGPL-2.1。第三方 Qt 為 LGPL-2.1、MinGW runtime 依其各自授權。執行細節同見
  下方 checklist。

簡言之：可以自由修改與再發布，但要保留授權與所有原始聲明、清楚標註自己的修改、
整體維持 GPL。

## 改動後要維護的文件（checklist）

任何程式碼改動：
- 在改到的既有檔案標頭，原著作權行下方加 `Copyright (C) 2026 Frank`
  （該檔第一次被本 fork 改到時加，之後不必重複），不要動到上游既有著作權行。
- 新增檔案：沿用相同的 GPL 標頭，著作權行寫 `Copyright (C) 2026 Frank`。
- commit 訊息清楚描述改了什麼、為什麼。
- 在 `Changelog.txt` 最上方「Release 1.0.x (fork)」段補一條說明。

改到使用者可見字串（UI 文字）：
- 更新 `src/lang/diskimager_en.ts`（基準）及其他語言檔，必要時標 unfinished。
- lrelease 產生對應 `.qm`（build 時 `.pro` 會自動跑 lrelease）。

發新版本（bump version）時，這幾處要同步：
- `src/DiskImager.pro` 的 VERSION
- `src/DiskImager.rc` 的 FILEVERSION / PRODUCTVERSION 及 FileVersion / ProductVersion 字串
- `Changelog.txt` 開新的 Release 段
- `README.md` 內提到版本之處
- 註：`setup.iss` 用 GetStringFileInfo 從 exe 自動讀版本，不必手改

發布 binary / GitHub Release 時：
- 一併提供對應原始碼（GPL 義務；push 到本 repo 即滿足）
- 隨附 `GPL-2`、`LGPL-2.1`、`License.txt`
- 若打包 Qt DLL，附 `LGPL-2.1` 文字並確保使用者能替換/重連結 Qt

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

改動後要更新哪些文件，見上方「改動後要維護的文件（checklist）」。以下是純技術注意：

- 第一個 commit 是未修改的上游基準，方便 diff 出本 fork 的改動；每個修正用獨立 commit。
- 換行：repo 內檔案原始為 LF，git blob 保留 LF。`.bat` 檔在 Windows 需要 CRLF，
  不要對它們做 LF 正規化。
- 含中文/CJK 的檔案（特別是 `src/lang/*.ts`）一律用 Edit / Write 工具修改；
  在 Windows 上避免用未指定 encoding 的 PowerShell `Get-Content -Raw` 做替換，會
  產生 mojibake。
- `*.pro.user`、`build*/`、`Release/`、`Debug/`、`*.exe`、`*.dll`、`*.qm` 已在
  `.gitignore` 內，不要 commit。
