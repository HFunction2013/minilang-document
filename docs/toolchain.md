# Toolchain & self-hosting

minilang is self-hosting: every C source has a minilang implementation.

## Repositories

| Repo | Contents |
|------|----------|
| `minilang-boot` | handwritten C: lexer, parser, bytecode compiler, stack VM, LLVM IR generator, milc serializer, native-method registry, CLI |
| `minilang-compiler` | the minilang sources (`compiler.mil`, `vm.mil`, `llvm_gen.mil`, `milc_io.mil`, `main.mil`) and tests |
| `minilang-syslib` | standard library (git submodule of both) |

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

## LLVM native linking

The native-executable backend uses **LLVM project binaries** (no Python
llvmlite):

```
llc -O2 -filetype=obj x.ll -o x.o
clang -O2 -no-pie x.o runtime.c -o x
```

Tools are resolved via `$LLVM_DIR/bin` first, then `PATH`; the linker can be
overridden with `$LLVM_LINKER`.

## CLI

```
minilang run <file>                # compile and run .mil, or run .milc
minilang run -e <file>.mil ext.c.. # boot VM: compile the .c extension(s), load
                                   # them and run (see docs/native.md)
minilang bytecode <file>           # disassembly
minilang dump-text <file>          # MINILANGBC text form
minilang llvm <file>               # emit .ll
minilang build -b <file>           # write .milc
minilang build -e <file>           # write native executable
minilang build -e <file> ext.c     # ... linking a user C extension
minilang build -e --dynamic <file> ext.c   # ... extension as a shared library
minilang repl                      # interactive REPL
minilang self-test                 # self-host check
```

## Windows

MSVC and MinGW-w64 are both supported: everything branches on `_WIN32`, and
MSVC builds through CMake (`cmake -B build && cmake --build build`).

* Executable discovery uses `GetModuleFileNameA`, `PATH` splits on `;`, and
  scratch files go to `%TEMP%` — no `/proc/self/exe`, `/tmp` or `:` separator.
* `run -e` packs `.c` sources + the generated ABI adapter + `runtime.c` into a
  `%TEMP%\ml_ext_<pid>.dll` and loads it with `LoadLibrary` (Windows has no
  `--export-dynamic` for an executable's symbols, so the DLL is self-contained).
* `build -e --dynamic` emits `<base>_extN.dll` plus an import library; if the
  linker cannot do import libraries it warns and falls back to static linking.
