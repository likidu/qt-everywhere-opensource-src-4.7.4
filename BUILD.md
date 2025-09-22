# Qt 4.7.4 — Build QtNetwork with TLS 1.1/1.2 (MinGW 4.4)

This repository includes a backport enabling TLS 1.1 and 1.2 in QtNetwork when running with OpenSSL 1.0.2u. This guide shows how to build only the QtNetwork library (and its dependencies) for the Qt Simulator using MinGW 4.4.

## Prerequisites

- MinGW 4.4: `C:\Symbian\QtSDK\mingw\bin`
- OpenSSL 1.0.2u headers and DLLs: `C:\Users\{your_name}\Repos\openssl-1.0.2u-symbian`
  - Expected layout:
    - Headers: `outinc\openssl\...` (this tree contains `opensslconf.h`)
      - Pass the parent include dir to the compiler: use `-I C:\\Users\\{your_name}\\Repos\\openssl-1.0.2u-symbian\\outinc`
    - DLLs: `ssleay32.dll`, `libeay32.dll` (for runtime)
    - Optional import libs (only for linked build): `libssl.dll.a`, `libcrypto.dll.a` under `lib\\` (or same folder)

## Notes

- This build uses Qt’s runtime OpenSSL resolver by default (dynamic). No link to OpenSSL import libs is required; headers are still needed to compile.
- The default SSL protocol is switched to `QSsl::AnyProtocol` in this tree.

## Environment Setup (PowerShell)

1. Open a “Qt 4.7 MinGW” PowerShell or normal PowerShell.
1. Set MinGW in PATH (put it first):

   ```powershell
   $env:PATH = "C:\Symbian\QtSDK\mingw\bin;" + $env:PATH
   ```

1. Set OpenSSL include/lib hints (use the parent of `openssl\`):

   ```powershell
   $env:OPENSSL_INCDIR = "C:\Users\{your_name}\Repos\openssl-1.0.2u-symbian\outinc"
   $env:OPENSSL_LIBDIR = "C:\Users\{your_name}\Repos\openssl-1.0.2u-symbian\out"
   ```

## Build

From repo root:

- `mingw32-make clean`

**Option A (dynamic OpenSSL)**

- `./configure -platform win32-g++ -opensource -confirm-license -nomake demos -nomake examples -no-webkit -openssl -I %OPENSSL_INCDIR`

**Option B (link to OpenSSL)**

What to pass:

- `-I C:\Users\{your_name}\Repos\openssl-1.0.2u-symbian\outinc`
- `-L C:\Users\{your_name}\Repos\openssl-1.0.2u-symbian\out`
- And set library NAMES (not paths). With your import libs libssleay32.a and libeay32.a: `cmd.exe: set OPENSSL_LIBS=-lssleay32 -leay32`

- `./configure -platform win32-g++ -opensource -confirm-license -nomake demos -nomake examples -no-webkit -openssl-linked -I %OPENSSL_INCDIR% -L %OPENSSL_LIBDIR%`

If your import libs are named libssl.dll.a and libcrypto.dll.a instead, use OPENSSL_LIBS=-lssl -lcrypto.
Don’t use -l with a path; only bare names after -l. Use -L for the directory.

- `mingw32-make -C src\corelib` as a prerequisite to build the network
- `mingw32-make -C src\network`

### To build other dep modules, which is often used for GUI apps

Build bootstrap and moc: already done earlier, but safe to rerun

- `mingw32-make -C src\tools\bootstrap`
- `mingw32-make -C src\tools\moc`

Build uic and rcc tools:

- `mingw32-make -C src\tools\uic`
- `mingw32-make -C src\tools\rcc`
- Optional (compat, sometimes used):` mingw32-make -C src\tools\uic3`

Build core dependencies and modules:

- `mingw32-make -C src\gui`
- `mingw32-make -C src\script`
- `mingw32-make -C src\xmlpatterns`
- `mingw32-make -C src\network`
- `mingw32-make -C src\declarative`

To force Debug explicitly: `mingw32-make -C src\corelib debug`

To add running jobs to speed up: `mingw32-make -C src\corelib -j8`

Use with Qt Simulator

- Put lib\QtNetwork4.dll (QtNetwork4d.dll) earlier on PATH than the Simulator's Qt, or next to the app using the Simulator.
- Ensure runtime DLLs `ssleay32.dll` and `libeay32.dll` from 1.0.2u are accessible (PATH or alongside the app).

## Troubleshooting

- If `QSslSocket::supportsSsl()` is false at runtime, your OpenSSL DLLs were not found or mismatched. Put 1.0.2u `ssleay32.dll` and `libeay32.dll` next to the app or first in PATH.
- If configure cannot find OpenSSL headers, confirm `%OPENSSL_INCDIR%\openssl\ssl.h` and `%OPENSSL_INCDIR%\openssl\opensslconf.h` exist and pass the absolute `-I` path to `configure`.
- If you see link errors when using `-openssl-linked`, confirm the MinGW import libs names and bitness match your toolchain.
- If you see compile errors about `openssl/opensslconf.h` or `krb5.h` from `openssl/kssl.h`:
  - Ensure you are pointing to OpenSSL’s installed headers (after `make install`) where `include\openssl\opensslconf.h` exists.
  - For builds without Kerberos, `opensslconf.h` should define `OPENSSL_NO_KRB5` to avoid `kssl.h` pulling `krb5.h`.
  - As a workaround, you can add `-DOPENSSL_NO_KRB5` to the build: run from repo root:
    - `set QMAKE_CXXFLAGS=%QMAKE_CXXFLAGS% -DOPENSSL_NO_KRB5` (cmd) then rebuild, or add `QMAKE_CXXFLAGS += -DOPENSSL_NO_KRB5` to `mkspecs\win32-g++\qmake.conf` and reconfigure.
