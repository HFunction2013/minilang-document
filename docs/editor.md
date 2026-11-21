# Editor support

## VS Code / Cursor extension

[`minilang-vscode`](https://github.com/HFunction2013/minilang-vscode) provides:

- Syntax highlighting for `.mil`
- A custom `.milc` bytecode viewer (Text Bytecode / Bytecode Assembly)
- Commands: run in terminal, REPL, bytecode, dump-text, generate C source, build -b/-e
- Settings: `minilang.path` (path to the `minilang` binary), `minilang.workingDirectory`

## Language server

[`minilang-server`](https://github.com/HFunction2013/minilang-server) is an
LSP server (stdio). The extension launches it automatically:

- **Diagnostics** — shells out to the `minilang` toolchain and maps
  `Parse error line N` / `Compile error` to squiggles as you type.
- **Hover** — docs for built-ins, syslib functions and C-native methods.
- **Completion** — built-ins, `math.*` / `string.*` / …, the 8 C-native
  methods (`sqrt`, `abs`, `time`, …), `native func` declarations in the open
  file, user-declared functions and variables, and keyword snippets
  (`native func`, `require ... from builtin`, `var`, `func`, `if`, `while`,
  `return`, `break`).
- **Goto definition** — jumps to a user `func`/`var`/`native func`
  declaration in the buffer, or to `syslib/<module>.mil` (and the matching
  function line) for module-qualified names like `math.min`.
- **Signature help** — `(` / `,` triggers the parameter list of built-ins,
  syslib functions, user natives and user functions, with the active
  parameter highlighted.
- **Document symbols** — outline of the buffer's functions, variables and
  native declarations (breadcrumbs / outline view).
- **Find references** — lists every whole-word occurrence of the name under
  the cursor in the current buffer.
- **Rename** — F2 renames the symbol under the cursor everywhere in the
  buffer (works on user functions, variables and native declarations).

Settings:

| Setting | Default | Purpose |
|---------|---------|---------|
| `minilang.useLanguageServer` | `true` | start the LSP server (off = built-in providers) |
| `minilang.serverPath` | bundled | path to `minilang-server/out/server.js` |

### Using the server outside VS Code

Any LSP client (Neovim, Zed, Emacs) can launch it:

```bash
npm install && npm run build   # in minilang-server
node out/server.js            # speaks LSP over stdio
```

Point the client at a built `minilang` binary so diagnostics work.
