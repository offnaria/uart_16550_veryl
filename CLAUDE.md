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
- Clock ports use type `clock`, reset ports use type `reset`; `always_ff { if_reset { ... } }` infers both automatically.
- `reset_type = "sync_low"` and `clock_type = "posedge"` are set in `Veryl.toml`.
- `let x: T = expr;` declares a combinational (wire) signal at module scope.
- Array of registers: `var mem: logic<W> [N];` — elements are writeable inside `always_ff`.

## Development Plan

Full design spec is in `SPEC.md`. Implementation is split into five source files built bottom-up:

| Step | File | Module | Status |
|---|---|---|---|
| 1 | `src/fifo16.veryl` | `Fifo16` | 🔲 in progress |
| 2 | `src/baud_gen.veryl` | `BaudGen` | 🔲 pending |
| 3 | `src/uart_tx.veryl` | `UartTx` | 🔲 pending |
| 4 | `src/uart_rx.veryl` | `UartRx` | 🔲 pending |
| 5 | `src/uart_16550.veryl` | `Uart16550` | 🔲 pending |

### Step 1 — `Fifo16`
16-entry first-word fall-through FIFO, parameterized by `WIDTH`.
- TX path uses `WIDTH=8`; RX path uses `WIDTH=11` (`{BI, FE, PE, data[7:0]}`).
- 5-bit gray-style pointers: bit 4 is wrap flag; full/empty detected by pointer comparison.
- Ports: `i_wr_en`, `i_wr_data`, `o_full`, `i_rd_en`, `o_rd_data`, `o_empty`, `o_count`.

### Step 2 — `BaudGen`
Generates a 1-cycle `o_en` pulse at 16× the baud rate.
- Input: 16-bit divisor `i_div` (from `{DLM, DLL}`), reload when divisor changes.
- Counter counts down from `i_div` to `1`; pulses `o_en` at rollover.
- Baud = `CLK_FREQ / (divisor × 16)`.

### Step 3 — `UartTx`
TX shift register driven by `BaudGen`'s 16× enable.
- Loads next byte from TX FIFO when shift register becomes idle.
- Outputs start bit (0), data LSB-first, optional parity, stop bit(s).
- Exposes `o_fifo_full` and `o_idle` for LSR `THRE`/`TEMT` bits.
- Word length, parity, stop bits configured via LCR fields passed as ports.

### Step 4 — `UartRx`
16× oversampling receiver; samples at tick 8 of each bit period.
- Start-bit detection on falling edge of `i_rxd`.
- Detects PE (parity error), FE (framing error), BI (break interrupt).
- Writes `{BI, FE, PE, data[7:0]}` into RX FIFO; signals OE when FIFO full.
- Word length, parity configured via LCR fields passed as ports.

### Step 5 — `Uart16550` (top level)
AXI4-Lite slave + register file + interrupt logic + loopback.
- AW/W accepted simultaneously; BRESP next cycle; AR→RDATA next cycle.
- Register decode: offset bits `[4:2]` select register; DLAB gates DLL/DLM vs RBR/THR/IER.
- Side effects on read: RBR advances RX FIFO, LSR clears OE, MSR clears DCTS.
- Interrupt priority encoder feeds IIR; `o_intr = MCR.OUT2 & any_pending`.
- Loopback: TX output → RX input, RTS → CTS; physical pins held inactive.
