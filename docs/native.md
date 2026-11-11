# Native methods (C)

A *native method* is a minilang function implemented in C instead of minilang.
Declare it with the `native` keyword and no body:

```minilang
require sqrt from builtin;     // declarations live in syslib/builtin.mil
println toString(sqrt(144));    // 12
```

## How it works

1. The lexer recognises the `native` keyword; the parser accepts
   `native func name(params);` (declaration only, no block).
2. The bytecode compiler registers the function with `is_native = 1`
   (address `-1`) and emits `OP_NATIVE` at call sites (instead of `OP_CALL`).
3. The VM pops the arguments, looks the name up in the C registry
   (`ml_native_lookup`) and calls the function pointer; the returned `Value` is
   pushed back.

## Calling convention

A native C function has the signature:

```c
#include "minilang.h"

Value my_native(VM *vm, Value *args, int argc) {
    int64_t n = args[0].as.integer;
    return make_int(n * 2);
}
```

Register it (typically at startup in `native.c`):

```c
ml_native_register("double", my_native);
```

Module-qualified names (e.g. `math.sqrt`) fall back to the unqualified suffix,
so a native registered as `sqrt` is reachable through any module.

## Built-in natives

Declared in `syslib/builtin.mil`, implemented in `native.c` (in
[`minilang-boot`](https://github.com/HFunction2013/minilang-boot)):

| Function | Description |
|-----------|-------------|
| `sqrt(n)` | floor of √n (n<0 → 0) |
| `abs(n)`  | absolute value |
| `time()`  | seconds since Unix epoch |
| `clock_ms()` | monotonic milliseconds since process start |
| `getenv(name)` | environment variable value |
| `cwrite(s)` | write a string directly to stdout |
| `exit(code)` | terminate the process |
| `csystem(cmd)` | run a shell command, return exit code |

Moving these out of minilang and into C shrinks the compiled `.milc` image
(no Newton-loop bytecode for `sqrt`, for example).

## User C extensions (run -e, build -e, OpaqueType)

Declare the functions in the `.mil` program:

```minilang
native func my_open(path);     // -> opaque handle
native func my_read(handle);   // -> int
```

There are two ways to supply the C implementation.

### Run straight from source (`run -e`)

The C boot VM compiles the `.c` file, loads it and registers the natives —
no separate build step:

```bash
minilang run -e prog.mil myext.c -- args...
minilang run -e prog.mil ext.so -- args...     # prebuilt shared library (.dll on Windows)
```

The generated program links `runtime.c` automatically; extra compiler/linker
flags can be injected with `LLVM_LINKER`.

### Build a native executable (`build -e`)

```bash
minilang build -e prog.mil myext.c            # static link (default)
minilang build -e --dynamic prog.mil myext.c  # extension as a shared library
```

`--dynamic` compiles each `.c` to `<base>_extN.so` (Linux/macOS) or
`<base>_extN.dll` (Windows, with an import library the executable links
against) so native code can be swapped without rebuilding the program.

### Calling convention

Implement each function as `Value ml_native_<name>(Value ...)` — base name, no
module prefix — in the extension C file, including the public API header
[`minilang_runtime.h`](https://github.com/HFunction2013/minilang-boot/blob/main/minilang_runtime.h):

```c
#include "minilang_runtime.h"   # -I points at the runtime dir automatically

Value ml_native_my_open(Value path) {
    FILE *f = fopen(ml_cstr(path), "r");
    return ml_opaque_create(f);            // wrap a C pointer (TAG_OPAQUE)
}

Value ml_native_my_read(Value handle) {
    FILE *f = (FILE *)ml_opaque_ptr(handle);  // unwrap, errors if not opaque
    return ml_mkint(f ? fgetc(f) : -1);
}
```

An alternative signature, used when an extension wants to see the raw VM
frame, is `MLValue ml_vm_native_<name>(void *vm, MLValue *args, int argc)`.
`run -e` accepts both: it probes the extension for `ml_vm_native_*` symbols
(or scans the source) and bridges between the 32-byte VM `Value` and the
16-byte runtime `Value` automatically.

> The **self-hosted** interpreter (`minilang run main.mil run ...`) cannot
> load C extensions; it prints an explicit error and exits with status 1. Use
> the boot binary (`minilang run -e ...`) or `build -e`.

### OpaqueType

`ml_opaque_create(ptr)` / `ml_opaque_ptr(v)` wrap and unwrap an arbitrary C
pointer (`FILE*`, buffers, structs) as a minilang `Value` (tag `4`). minilang
cannot inspect the payload; it only:

- stores / passes the value (variables, args, arrays),
- compares by pointer identity (`==` / `!=`),
- treats non-NULL as truthy in `if`,
- prints it as `<opaque>`.

Other runtime helpers: `ml_nil`, `ml_is_nil/int/str/arr/opaque`,
`ml_int`, `ml_mkint`, `ml_cstr`, `ml_mkstr`, `ml_mkarray(n)`, `ml_array_len(v)`.
See `tests/ext/` (`file_counter.c` + `opaque_test.mil`) and `ext_test.sh` in
[`minilang-compiler`](https://github.com/HFunction2013/minilang-compiler) for a
working example.

### Windows notes

On Windows (MSVC or MinGW-w64), `run -e` packs the `.c` sources, the generated
ABI adapter and `runtime.c` into a single `%TEMP%\ml_ext_<pid>.dll`, loads it
with `LoadLibrary` and registers its `dllexport`ed entry points. Bundling
`runtime.c` keeps the DLL self-contained, because Windows has no
`--export-dynamic` equivalent for symbols an executable exports. `build -e
--dynamic` writes `<base>_extN.dll` plus an import library; if the linker
cannot do import libraries it warns and falls back to static linking.
