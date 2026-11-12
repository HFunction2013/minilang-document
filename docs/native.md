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

### Precompile first (no on-the-fly C compilation)

The toolchain **never compiles extension C on the fly**. You build the C
implementation yourself, then pass the resulting artefact to minilang:

```bash
# 1. Precompile your extension (object file for static linking, shared lib for loading)
cc -O2 -fPIC -I<minilang-boot> -c myext.c -o myext.o
cc -O2 -fPIC -shared -I<minilang-boot> myext.c -o myext.so   # myext.dll on Windows

# 2a. Load the prebuilt shared library while interpreting
minilang run -e prog.mil myext.so -- args...

# 2b. Or link a precompiled object/archive/shared library into a native executable
minilang build -e prog.mil myext.o
```

Passing a raw `.c` source is rejected: the boot backend reports an error that
extensions must be precompiled. The `--dynamic` flag is still accepted for
backwards compatibility but no longer changes the link mode — a prebuilt shared
library is already dynamic.

### Calling conventions

There are two C signatures; the loader probes the symbols it finds.

**Packed ABI** (preferred for extensions), base name without module prefix,
using the public API header
[`minilang_runtime.h`](https://github.com/HFunction2013/minilang-boot/blob/main/minilang_runtime.h):

```c
#include "minilang_runtime.h"   // -I points at the runtime dir automatically

Value ml_native_my_open(Value path) {
    FILE *f = fopen(ml_cstr(path), "r");
    return ml_opaque_create(f);            // wrap a C pointer (TAG_OPAQUE)
}

Value ml_native_my_read(Value handle) {
    FILE *f = (FILE *)ml_opaque_ptr(handle);  // unwrap, errors if not opaque
    return ml_mkint(f ? fgetc(f) : -1);
}
```

**VM-style ABI**, when the extension wants the raw VM frame:
`MLValue ml_vm_native_<name>(void *vm, MLValue *args, int argc)`.

`run -e` registers `ml_vm_native_*` symbols directly. Packed `ml_native_*`
symbols are wrapped by an internal trampoline that converts between the 32-byte
VM `Value` and the 16-byte runtime `Value` automatically.

> Bridge limits: the packed trampoline supports arity **0–3** arguments; calls
> with more arguments report an error and return nil. Array arguments crossing
> the ABI boundary are converted best-effort and are **not** deep-converted.
> Nil, int, string and opaque round-trip reliably.

> The **self-hosted** interpreter supports `run -e` as well:
> `minilang run main.mil run -e prog.mil ext.so ... [-- progargs]` loads the
> prebuilt library into the host VM and dispatches the declared natives through
> the host registry. Arguments after `--` are visible to the program as `argv()`.
> The two entry points produce identical output.

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

On Windows (MSVC or MinGW-w64) you precompile the extension into a `.dll` (or
`.obj`/static archive) yourself, exactly as on Unix. The loader uses
`LoadLibrary` for the shared library; there is no step that packs `.c` sources
and `runtime.c` into a temporary process DLL.

