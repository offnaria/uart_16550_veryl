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
- Standard library is under `$std` namespace; no dependency declaration needed. Example: `inst u: $std::fifo #(WIDTH: 8, DEPTH: 16) (i_clk: _, ...);`

## Testing Policy

Each module has a dedicated test file in `test/`. Write and run its test before moving to the next module. Run all tests with `veryl test --sim verilator`.

### Test file conventions

- File: `test/test_<module>.veryl`, one `#[test(name)]` module per scenario
- Use `inst u_clock: $tb::clock_gen;` for clock, `u_clock.next(N)` to advance N cycles
- Use `$assert(cond)` for checks, `$finish()` to end simulation
- Use small `CLK_FREQ` (e.g. 16) with divisor=1 for fast simulation (1 baud bit = 16 clocks)
- Test files are included via `sources = ["src", "test"]` in `Veryl.toml`
- When instantiating a DUT with `clock`/`reset` typed ports, omit those ports and annotate with `#[allow(missing_port)]` — the native test runner connects them implicitly via `$tb::clock_gen`

### Test coverage per module

| Test file | Module under test | What to verify |
|---|---|---|
| `test/test_baud_gen.veryl` | `BaudGen` | `o_en` pulse period matches `divisor × 16` clocks; divisor change takes effect immediately |
| `test/test_uart_tx.veryl` | `UartTx` | Correct bit stream on `o_txd`: start=0, data LSB-first, parity (if enabled), stop=1; `o_idle` and `o_fifo_full` timing |
| `test/test_uart_rx.veryl` | `UartRx` | Valid frame → correct data + no errors; framing error; parity error; overrun when FIFO full |
| `test/test_uart_16550.veryl` | `Uart16550` | AXI write DLL/DLM + LCR; AXI write THR → verify TX bitstream; drive RXD → AXI read RBR; loopback mode; interrupt assertion/deassertion |

## Development Plan

Full design spec is in `SPEC.md`. FIFOs use `$std::fifo` (no custom FIFO needed). Implementation is split into four source files built bottom-up. **Each step includes writing and passing the corresponding test before proceeding.**

| Step | Src file | Test file | Status |
|---|---|---|---|
| 1 | `src/baud_gen.veryl` | `test/test_baud_gen.veryl` | ✅ done |
| 2 | `src/uart_tx.veryl` | `test/test_uart_tx.veryl` | ✅ done |
| 3 | `src/uart_rx.veryl` | `test/test_uart_rx.veryl` | 🔲 pending |
| 4 | `src/uart_16550.veryl` | `test/test_uart_16550.veryl` | 🔲 pending |

### `$std::fifo` usage
- TX FIFO: `$std::fifo #(WIDTH: 8, DEPTH: 16)` — `i_clear` maps to FCR.TXRST
- RX FIFO: `$std::fifo #(WIDTH: 11, DEPTH: 16)` — stores `{BI, FE, PE, data[7:0]}`; `i_clear` maps to FCR.RXRST
- `o_word_count` is `logic<5>` (0–16); compare against RX trigger level for RDA interrupt
- `DATA_FF_OUT = true` (default): `o_data` is registered — capture it in the same cycle ARVALID is accepted, before asserting `i_pop`

### Step 1 — `BaudGen`
Generates a 1-cycle `o_en` pulse at 16× the baud rate.
- Input: 16-bit divisor `i_div` (from `{DLM, DLL}`), reload when divisor changes.
- Counter counts down from `i_div` to `1`; pulses `o_en` at rollover.
- Baud = `CLK_FREQ / (divisor × 16)`.

### Step 2 — `UartTx`
TX shift register driven by `BaudGen`'s 16× enable.
- Loads next byte from TX FIFO when shift register becomes idle.
- Outputs start bit (0), data LSB-first, optional parity, stop bit(s).
- Exposes `o_fifo_full` and `o_idle` for LSR `THRE`/`TEMT` bits.
- Word length, parity, stop bits configured via LCR fields passed as ports.

### Step 3 — `UartRx`
16× oversampling receiver; samples at tick 8 of each bit period.
- Start-bit detection on falling edge of `i_rxd`.
- Detects PE (parity error), FE (framing error), BI (break interrupt).
- Writes `{BI, FE, PE, data[7:0]}` into RX FIFO; signals OE when FIFO full.
- Word length, parity configured via LCR fields passed as ports.

### Step 4 — `Uart16550` (top level)
AXI4-Lite slave + register file + interrupt logic + loopback.
- AW/W accepted simultaneously; BRESP next cycle; AR→RDATA next cycle.
- Register decode: offset bits `[4:2]` select register; DLAB gates DLL/DLM vs RBR/THR/IER.
- Side effects on read: RBR advances RX FIFO, LSR clears OE, MSR clears DCTS.
- Interrupt priority encoder feeds IIR; `o_intr = MCR.OUT2 & any_pending`.
- Loopback: TX output → RX input, RTS → CTS; physical pins held inactive.
