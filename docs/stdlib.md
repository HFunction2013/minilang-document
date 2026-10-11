# Standard library

Modules live in the `syslib/` directory (a git submodule,
[`minilang-syslib`](https://github.com/HFunction2013/minilang-syslib)) or next
to the script. Loaded with `require`.

## require

```minilang
require math;                 // namespace access: math.cos(60)
require cos from math;        // selective import: cos(60)
require cos, sin from math;   // multiple selective imports
require cos from math in syslib;   // force search location (syslib | cwd)
```

Search order: script directory first, then `syslib/`. The executable resolves
`syslib/` relative to itself, so programs run from any working directory.

## Modules

| Module | Provides |
|--------|----------|
| `builtin` | declarations for C-native methods (`sqrt`, `abs`, `time`, `clock_ms`, `getenv`, `cwrite`, `exit`, `csystem`) |
| `math` | `min, max, clamp, pow, gcd, lcm, isEven, isOdd, factorial, fib, cos, sin, pi` |
| `string` | `isEmpty, startsWith, endsWith, contains, repeat, reverse, split, lines, words, join, indexOf, toUpper, toLower, trim, …` |
| `list` | `range, copy, push, pop, first, last, slice, concat, sum, min, max, contains, …` |
| `sort` | `ints, intsDesc, strings, insert, bsearch, bsearchStr, isSorted` (in place) |
| `json` | tagged-pair JSON: `parse, stringify, pretty, get, set, obj, arr, …` |
| `io` | `read, write, append, exists, readLines, pathJoin, basename, dirname, ext, stem, withExt` |
| `rand` | deterministic LCG: `seed, next, int, range, chance, pick, shuffle` |
| `word` | example showing nested `require` |

## JSON representation

minilang has no hash type, so JSON values are tagged pairs `[type, payload]`:

| type | meaning |
|------|---------|
| 0 | null |
| 1 | bool |
| 2 | number (int only) |
| 3 | string |
| 4 | array |
| 5 | object `[5, [keys, values]]` |
