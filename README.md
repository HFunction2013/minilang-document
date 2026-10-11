# minilang documentation

Documentation for the **minilang** programming language — a tiny, self-hosting,
C-like language with a bytecode VM and an LLVM native backend.

This content is licensed under **Creative Commons Attribution-ShareAlike 4.0
International (CC BY-SA 4.0)**. See the [`LICENSE`](LICENSE) file. Code examples
may be reused under the same terms.

## Repository layout

| Doc | Contents |
|-----|----------|
| [docs/language.md](docs/language.md) | Syntax, types, operators, statements |
| [docs/stdlib.md](docs/stdlib.md) | `require` module system and the bundled modules |
| [docs/native.md](docs/native.md) | C-native methods (`native func ...`) |
| [docs/toolchain.md](docs/toolchain.md) | Boot compiler, self-hosting, LLVM linking |
| [docs/editor.md](docs/editor.md) | VS Code extension + language server |

## The minilang ecosystem

| Repo | Role |
|------|------|
| [`minilang-compiler`](https://github.com/HFunction2013/minilang-compiler) | Self-hosted `.mil` sources, tests |
| [`minilang-boot`](https://github.com/HFunction2013/minilang-boot) | Handwritten C bootstrap compiler + VM |
| [`minilang-syslib`](https://github.com/HFunction2013/minilang-syslib) | Standard library modules (submodule) |
| [`minilang-server`](https://github.com/HFunction2013/minilang-server) | Language Server (LSP) |
| [`minilang-vscode`](https://github.com/HFunction2013/minilang-vscode) | VS Code / Cursor extension |

## Quick example

```minilang
func fib(n) {
    if (n < 2) { return n; }
    return fib(n - 1) + fib(n - 2);
}

func main() {
    var i = 0;
    while (i < 10) {
        println toString(fib(i));
        i = i + 1;
    }
    return 0;
}
```
