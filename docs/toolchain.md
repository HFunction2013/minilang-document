# Toolchain & self-hosting

minilang is self-hosting: every C source has a minilang implementation.

## Repositories

| Repo | Contents |
|------|----------|
| `minilang-boot` | handwritten C: lexer, parser, bytecode compiler, stack VM, C source generator, milc serializer, native-method registry, CLI |
| `minilang-compiler` | the minilang sources (`compiler.mil`, `vm.mil`, `milc_io.mil`, `main.mil`) and tests |
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

## C native build

The native-executable backend emits a **C source file** (`<base>.c`) and
compiles it at `-O2` (MSVC: `/O2`) with the system compiler:

```
# GCC / Clang / g++ (Unix and Windows clang/MinGW)
cc -O2 -I<runtime dir> <base>.c runtime.c [prebuilt extensions] -o <bin>

# MSVC (cl.exe)
cl /O2 /I<runtime dir> <base>.c runtime.c [prebuilt extensions] /Fe<bin>
```

The driver is resolved from `$CC` (Unix default `cc`, Windows default `clang`)
and classified by basename: `cl`/`cl.exe` selects the MSVC dialect
(`/O2 /I /Fe`), everything else (`cc`, `gcc`, `g++`, `clang`, `clang++`) uses
the GCC/Clang dialect. The runtime and generated source are C++-clean so a C++
driver such as `g++` can build them; no LLVM/llc/llvmlite step is involved.

## CLI

```
minilang run <file>                # compile and run .mil, or run .milc
minilang run -e <file>.mil ext.so  # load a prebuilt shared library and run
                                   # (boot VM and self-hosted `run main.mil run -e`
                                   # are equivalent; see docs/native.md)
minilang bytecode <file>           # disassembly
minilang dump-text <file>          # MINILANGBC text form
minilang c <file>                  # emit C source to <base>.c
minilang build -b <file>           # write .milc
minilang build -e <file>           # write native executable
minilang build -e <file> ext.o     # ... linking a prebuilt object/archive/shared lib
minilang repl                      # interactive REPL
minilang self-test                 # self-host check
```

## Windows

MSVC and MinGW-w64 are both supported: everything branches on `_WIN32`, and
MSVC builds through CMake (`cmake -B build && cmake --build build`).

* Executable discovery uses `GetModuleFileNameA`, `PATH` splits on `;`, and
  scratch files go to `%TEMP%` — no `/proc/self/exe`, `/tmp` or `:` separator.
* Extensions are precompiled to a `.dll` (or `.obj`/static archive) yourself;
  `run -e` loads the DLL with `LoadLibrary`. The toolchain does not bundle
  `.c` sources into a temporary process DLL.
