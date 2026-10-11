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
minilang run <file>          # compile and run .mil, or run .milc
minilang bytecode <file>     # disassembly
minilang dump-text <file>   # MINILANGBC text form
minilang llvm <file>         # emit .ll
minilang build -b <file>     # write .milc
minilang build -e <file>     # write native executable
minilang repl                # interactive REPL
minilang self-test           # self-host check
```
