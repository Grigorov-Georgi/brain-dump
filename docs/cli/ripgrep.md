# ripgrep (rg) cheatsheet

## Core flags

### Case / matching mode

- **`-i`, `--ignore-case`** — Ignore case. `rg -i fast` matches `fast`, `fASt`, `FAST`, etc.
- **`-F`, `--fixed-strings`** — Treat the pattern as a literal string (no regex).
- **`-w`, `--word-regexp`** — Match only on word boundaries (`\b(?:PATTERN)\b`).

### Output / reporting

- **`-c`, `--count`** — Report the count of matched lines.
- **`--files`** — Print the files that `rg` would search, but don’t search them.
- **`-r REPL`, `--replace REPL`** — Replace matches in the output only (does not modify files).

## Globs / file filtering

- **`-g 'GLOB'`** — Restrict search to files matching a glob.

```bash
rg clap -g '*.toml'
```

## Command examples

Replace in output (not in files):

```bash
rg fast README.md --replace FAST
```

Searches:

```bash
rg 'fn run' -trust
rg clap -Trust
rg clap -g '*.toml'
```

## Note on `-trust` / `-Trust`

`-trust` and `-Trust` are not valid ripgrep flags in the manpage/docs. If they come from an alias or plugin (or you meant something else), check:

```bash
rg --help | rg -i trust
rg --help | rg -i "ignore|hidden|no-ignore|unrestricted|threads|pcre|type|glob"
```

If you know what you intended those to do, replace them with the correct ripgrep flags.
