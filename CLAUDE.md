# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A UART 16550 implementation written in [Veryl](https://veryl-lang.org/), a modern hardware description language that transpiles to SystemVerilog. Veryl version: 0.20.1.

## Common Commands

```bash
veryl fmt          # Format all source files
veryl check        # Analyze / lint the project
veryl build        # Transpile Veryl → SystemVerilog (output in target/)
veryl clean        # Remove build artifacts
veryl test         # Run all tests (requires a simulator)
veryl test -t <name>  # Run tests matching a substring
```

### Simulator options for `veryl test`

Pass `--sim` to choose a simulator: `verilator` (default), `vcs`, `dsim`, or `vivado`. Example:

```bash
veryl test --sim verilator
veryl test --sim verilator --wave   # also dump waveforms
```

## Project Layout

- `src/` — Veryl source files (`.veryl`)
- `target/` — generated SystemVerilog output (gitignored)
- `dependencies/` — fetched package dependencies (gitignored)
- `Veryl.toml` — project manifest (name, version, source paths, build target)

## Veryl Language Notes

- Module definitions use `module`, interfaces use `interface`, packages use `package`.
- Tests are written inline using `#[test]` attributes and the `veryl_builtin` test infrastructure.
- `#[ifdef NAME]` / `-D NAME` control conditional compilation.
- The build target is set to `{type = "directory", path = "target"}` in `Veryl.toml`.
