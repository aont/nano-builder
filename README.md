# GNU nano for MSYS2 UCRT64

This repository packages GNU nano 9.2 as a native UCRT64 application for
MSYS2. It contains the MSYS2 `PKGBUILD`, a Windows compatibility patch, and a
manually triggered GitHub Actions workflow that builds, validates, and
publishes the package.

## Repository contents

| Path | Purpose |
| --- | --- |
| `mingw-w64-nano/PKGBUILD` | Package metadata and build instructions for MSYS2 UCRT64. |
| `mingw-w64-nano/nano-9.2-ucrt64.patch` | Windows portability and wide-character input changes applied to nano 9.2. |
| `.github/workflows/build-nano-ucrt64.yml` | Manual build, validation, artifact upload, and GitHub Release workflow. |
| `error-analysis.md` | Investigation of the remaining Japanese IME input issue. |
| `fixplan.md` | Proposed short- and long-term fixes for Unicode console input. |

## Requirements

Builds target the **UCRT64** environment. Install
[MSYS2](https://www.msys2.org/) and open an MSYS2 UCRT64 shell, then install
the build tools:

```sh
pacman -Syu
pacman -S --needed base-devel git
```

The remaining MinGW dependencies are declared by the `PKGBUILD` and can be
installed automatically by `makepkg-mingw`.

## Build locally

From the repository root, run:

```sh
cd mingw-w64-nano
MINGW_ARCH=ucrt64 makepkg-mingw \
  --cleanbuild \
  --syncdeps \
  --noconfirm \
  --force
```

The resulting pacman package is written to `mingw-w64-nano/` with a name like
`mingw-w64-ucrt-x86_64-nano-9.2-1-any.pkg.tar.zst`.

## Install

Install a locally built package or a package downloaded from a GitHub Release:

```sh
pacman -U mingw-w64-ucrt-x86_64-nano-9.2-1-any.pkg.tar.zst
```

Confirm that the executable starts:

```sh
/ucrt64/bin/nano.exe --version
```

## Automated release workflow

The **Build GNU nano pacman package for MSYS2 UCRT64** workflow is started
manually with `workflow_dispatch`. It:

1. verifies the package sources and checksums;
2. builds the UCRT64 pacman package;
3. checks the package contents and installs it;
4. verifies that `nano.exe` does not depend on `msys-2.0.dll`;
5. uploads the package and its SHA-256 checksum; and
6. creates an immutable GitHub Release tagged `v<package version>.<pkgrel>`.

Before rebuilding an already published version, increment `pkgrel` in the
`PKGBUILD`; the workflow refuses to overwrite an existing release tag.

## Known Unicode input limitation

Japanese IME input can remain unresponsive when ncurses uses its native
Windows console (`win32con`) input path. Unicode code points may be classified
as ncurses special-key codes before nano's UTF-8 conversion runs. See
[`error-analysis.md`](error-analysis.md) for the traced input path and
[`fixplan.md`](fixplan.md) for the proposed remediation and test plan.

This limitation is terminal-driver dependent, so behavior can differ between
the native Windows console and VT/ConPTY-based terminals.

## Updating the package

When updating nano or changing the patch:

1. update `pkgver` and/or `pkgrel` in `mingw-w64-nano/PKGBUILD`;
2. update the source and patch entries as necessary;
3. regenerate every changed SHA-256 value in `sha256sums`;
4. build locally in an MSYS2 UCRT64 shell; and
5. verify the package and run `nano.exe --version` before dispatching the
   release workflow.

GNU nano is distributed under the GPL-3.0-or-later license, as declared by the
package metadata.
