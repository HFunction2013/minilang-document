# Toolchain & self-hosting

minilang is self-hosting: every C source has a minilang implementation.

## Repositories

| Repo | Contents |
|------|----------|
| `minilang-boot` | handwritten C: lexer, parser, bytecode compiler, stack VM, C source generator, milc serializer, native-method registry, CLI |
| `minilang-compiler` | the minilang sources (`compiler.mil`, `vm.mil`, `milc_io.mil`, `main.mil`) and tests |
| `minilang-syslib` | standard library (git submodule of both) |

## Install

Each repo ships an installer under `scripts/` that fetches the sources,
compiles on site, lays down a user prefix, adds `bin` to `PATH`, and runs the
relevant test suite.

Unix (Linux / macOS):

```bash
git clone https://github.com/HFunction2013/minilang-compiler
cd minilang-compiler
bash scripts/install.sh          # full toolchain + documentation; prefix ~/.local
```

Windows (PowerShell):

```powershell
git clone https://github.com/HFunction2013/minilang-compiler
cd minilang-compiler
.\scripts\install.ps1            # prefix %LOCALAPPDATA%\Programs\minilang
```

The `minilang-boot` installer (`scripts/install.sh` / `install.ps1`) installs
only the boot VM, C runtime, headers and standard library.

Installed layout under `$PREFIX` (default `~/.local`; Windows
`%LOCALAPPDATA%\Programs\minilang`):

- `bin/minilang` (or `bin\minilang.cmd`) — wrapper put on `PATH`
- `lib/minilang/bin/minilang(.exe)` — boot binary
- `lib/minilang/` — runtime C sources/headers, self-hosted `.mil` sources,
  `syslib/`
- `share/doc/minilang/docs/` — user documentation

Set `MINILANG_PREFIX` to change the prefix. The installer appends
`$PREFIX/bin` to your shell profile (Unix) or user `PATH` (Windows); reopen
the terminal afterward.

## Bootstrapping

```bash
# 1. build the C boot compiler
git clone https://github.com/HFunction2013/minilang-boot
make -C minilang-boot

# 2. self-hosted sources
git clone https://github.com/HFunction2013/minilang-compiler
cd minilang-compiler
git submodule update --init
make MINILANG_BOOT=$PWD/../minilang-boot/minilang
```

## Self-hosting verification

```
minilang-boot/minilang build -e compiler.mil   # C boot compiles compiler.mil -> native exe A
./compiler dump-text compiler.mil              # A compiles compiler.mil -> bytecode B
minilang-boot/minilang dump-text compiler.mil  # C boot compiles compiler.mil -> bytecode A'
diff A' B                                      # must be identical
```

`./bootstrap_test.sh` automates this.

## C native build

The native-executable backend emits a **C source file** (`<base>.c`) and
compiles it at `-O2` (MSVC: `/O2`) with the system compiler:

```
# GNU / Clang (Unix)
cc -O2 -I<runtime dir> <base>.c runtime.c [prebuilt extensions] -o <bin>

# MSVC (cl.exe, Windows)
cl /O2 /I<runtime dir> <base>.c runtime.c [prebuilt extensions] /Fe<bin>
```

The driver is resolved from `$CC` (Unix default `cc`, Windows default `cl`)
and classified by basename: `cl`/`cl.exe` selects the MSVC dialect
(`/O2 /I /Fe`), everything else (`cc`, `gcc`, `g++`, `clang`, `clang++`, ...)
uses the GCC/Clang dialect. C++ drivers are forced to compile the translation
unit as C (`-x c`) so the packed ABI stays unmangled. The runtime and generated
source are C++-clean.

Three driver classes are supported: **MSVC** (`cl`, Windows only), **GCC/G++**
(`gcc`/`g++`, including versioned aliases such as `g++-15`), and **Clang**
(`clang`/`clang++`). CI covers the per-OS feasible combinations: Linux
gcc / g++ / clang, macOS clang / g++-15, Windows cl / clang (MSVC ABI) and
MinGW g++. The MinGW g++ job builds the boot host with the MinGW/Ninja
toolchain, which exports all host symbols and emits a GNU import library
(`libminilang.dll.a`) for the prebuilt extension DLL; the cl/clang jobs keep
the VS generator and its MSVC `minilang.lib`. MSVC/cl has no macOS or Linux
build. No LLVM/llc/llvmlite step is involved.

## CLI

```
minilang run <file>                # compile and run .mil, or run .milc
minilang run -e <file>.mil ext.so  # load a prebuilt shared library and run
                                   # (boot VM and self-hosted `run main.mil run -e`
                                   # are equivalent; see docs/native.md)
minilang bytecode <file>           # disassembly
minilang dump-text <file>          # MINILANGBC text form
minilang c <file>                  # emit C source to <base>.c
minilang genheader <file>          # emit packed-ABI <base>.h + stub <base>.c for native funcs
minilang compileheader <file>      # precompile the stub into <base>.<so|dylib|dll>
minilang build -b <file>           # write .milc
minilang build -e <file>           # write native executable
minilang build -e <file> ext.o     # ... linking a prebuilt object/archive/shared lib
minilang repl                      # interactive REPL
minilang self-test                 # self-host check
```

## Windows

MSVC and MinGW-w64 are both supported: everything branches on `_WIN32`.
MSVC/clang build the host through CMake with the VS generator
(`cmake -B build && cmake --build build --config Release`); the MinGW g++ job
configures CMake with the Ninja generator and gcc, linking
`--export-all-symbols --out-implib=libminilang.dll.a` so MinGW extension DLLs
can consume a GNU import library.

* Executable discovery uses `GetModuleFileNameA`, `PATH` splits on `;`, and
  scratch files go to `%TEMP%` — no `/proc/self/exe`, `/tmp` or `:` separator.
* Extensions are precompiled to a `.dll` (or `.obj`/static archive) yourself;
  `run -e` loads the DLL with `LoadLibrary`. The toolchain does not bundle
  `.c` sources into a temporary process DLL.
