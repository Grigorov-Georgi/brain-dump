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

## Type: `-t rust` / `-T rust`

- **`-t rust`, `-trust`** — Match only in Rust files (`-t` = `--type`).
- **`-T rust`, `-Trust`** — Negation: exclude Rust files (`-T` = `--type-not`).
