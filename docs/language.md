# Language reference

minilang source files use the `.mil` suffix; compiled bytecode uses `.milc`
(magic `!milc`).

## Types

| Type | Description |
|------|-------------|
| `int` | 64-bit signed integer |
| `string` | UTF-8-ish byte string |
| `array` | fixed-size array of values (reference semantics) |
| `nil` | empty value (produced by C natives; **there is no `nil` literal** — `nil` is not a keyword) |

There are no floats; trig returns integers scaled by 1000.

## Keywords

`var`, `func`, `if`, `else`, `while`, `return`, `break`, `print`, `println`,
`require`, `from`, `in`, `native`. (`true` / `false` / `nil` are **not**
keywords; they are ordinary identifiers.)

## Comments

```minilang
// line comment
/* block comment,
   spans lines */
```

## Declarations

```minilang
var GLOBAL = 42;          // global variable

func add(a, b) {          // function (recursive)
    var local = a + b;    // local variable
    return local;
}
```

A function declared with `native` has no body and is implemented in C — see
[native.md](native.md).

## Control flow

```minilang
if (x < 0) {
    println "negative";
} else {
    println "non-negative";
}

while (i < 10) {
    if (i == 5) { break; }
    i = i + 1;
}
```

## Operators

- Arithmetic: `+  -  *  /  %`
- Comparison: `==  !=  <  >  <=  >=`
- Logical: `&&  ||  !` (no short-circuit; use nested `if` to avoid type errors)
- Unary: `-` (negate), `!` (logical not)
- `+` also concatenates strings.

## Arrays and strings

```minilang
var a = array(3, 0);   // [0, 0, 0]
a[0] = 100;
var s = "Hello" + ", world";
var c = s[0];          // charAt
var sub = substr(s, 0, 5);
```

## Built-in functions

`print`, `println`, `len`, `charAt`, `substr`, `toString`, `toInt`, `strcmp`,
`readAll`, `array`, `argc`, `argv`, `readFile`, `fileExists`, `writeFile`,
`system`, `readLine`, `readFileBytes`, `writeFileBytes`, `chr`.

## Modules (require)

```minilang
require sqrt, abs from builtin;   // selective import, unqualified names
require json in syslib;           // whole module as namespace: json.parse(...)
require mylib in cwd;             // relative to the source file's directory
```

`in syslib` resolves against the toolchain's `syslib/` directory (the
[`minilang-syslib`](https://github.com/HFunction2013/minilang-syslib)
submodule), `in cwd` against the directory of the importing file. Imported
functions may also be called through their module namespace.

## Entry point

Execution starts at `func main()`. Command-line arguments are available via
`argc()` / `argv(i)`.
