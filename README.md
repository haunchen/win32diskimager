# Win32 Disk Imager (Unofficial Fork)

> This is an **unofficial community fork** of Win32 Disk Imager, maintained by
> [@haunchen](https://github.com/haunchen). It is **not** affiliated with or
> endorsed by the original ImageWriter developers. For the original project, see
> the upstream sources linked below.

A utility for Microsoft Windows to read and write raw disk images (`.img`) to
removable SD and USB storage devices.

## ⚠️ Warning — read this first

This tool writes directly to physical disks. **Selecting the wrong device will
permanently destroy all data on it.** Always double-check the target drive
letter before reading or writing. It cannot write CD-ROMs, and USB floppy
drives are not supported.

This software is provided **without any warranty**. The authors and maintainers
take no responsibility for data loss or any other damage. Use at your own risk.

## Features

- Write a raw image file to an SD card or USB device.
- Read a device back into an image file.
- Verify an image file against a device.
- MD5 / SHA1 / SHA256 checksums.
- Option to read only allocated partitions.
- Remembers the last used folder.
- Translations for several languages (English, German, Spanish, French,
  Italian, Japanese, Korean, Dutch, Polish, Tamil, Simplified Chinese,
  Traditional Chinese).

## Download

Pre-built releases (when available) are published on the
[Releases page](https://github.com/haunchen/win32diskimager/releases).

The program requires administrator privileges to access raw devices.

## Building from source

Requirements:

- Qt 5.7 with MinGW 5.3 (or a compatible Qt 5 / MinGW toolchain).

Quick build with the bundled scripts:

```bat
compile.bat   :: runs qmake + mingw32-make in src/
clean.bat     :: cleans build artifacts
```

Or open `src/DiskImager.pro` in Qt Creator and build. Because the application
requires administrator rights to run, launch Qt Creator as administrator when
debugging. See `DEVEL.txt` for more detail.

## Changes in this fork

See [`Changelog.txt`](Changelog.txt) for the list of changes made in this fork
relative to upstream 1.0.0.

## License

Win32 Disk Imager is licensed under the **GNU General Public License, version 2
or (at your option) any later version** (`GPL-2.0-or-later`). The full text is
in [`LICENSE`](LICENSE) (a copy of [`GPL-2`](GPL-2)).

This fork preserves the original license and all upstream copyright notices, as
required by the GPL. It is **not** relicensed.

Third-party components:

- The Qt library is used under the GNU Lesser General Public License, version
  2.1 (`LGPL-2.1`); see [`LGPL-2.1`](LGPL-2.1) and <https://www.qt.io/>.
- The MinGW runtime is used and available at <https://www.mingw-w64.org/>.

If you distribute compiled binaries together with Qt DLLs, you must comply with
the LGPL (ship the LGPL text and allow users to replace/relink the Qt
libraries).

## Credits

- Original version developed by Justin Davis `<tuxdavis@gmail.com>`.
- Maintained upstream by the ImageWriter developers:
  <https://sourceforge.net/projects/win32diskimager/>
- Upstream source repository:
  <https://sourceforge.net/p/win32diskimager/code/ci/master/tree/>
- This unofficial fork is maintained by Frank ([@haunchen](https://github.com/haunchen)).
