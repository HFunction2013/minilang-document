# Editor support

## VS Code / Cursor extension

[`minilang-vscode`](https://github.com/HFunction2013/minilang-vscode) provides:

- Syntax highlighting for `.mil`
- A custom `.milc` bytecode viewer (Text Bytecode / Bytecode Assembly)
- Commands: run in terminal, REPL, bytecode, dump-text, LLVM IR, build -b/-e
- Settings: `minilang.path` (path to the `minilang` binary), `minilang.workingDirectory`

## Language server

[`minilang-server`](https://github.com/HFunction2013/minilang-server) is an
LSP server (stdio). The extension launches it automatically:

- **Diagnostics** — shells out to the `minilang` toolchain and maps
  `Parse error line N` / `Compile error` to squiggles as you type.
- **Hover** — docs for built-ins, syslib functions and C-native methods.
- **Completion** — built-ins, `math.*` / `string.*` / … and native methods.

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
