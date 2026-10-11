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
