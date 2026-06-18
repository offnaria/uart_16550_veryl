# uart_16550_veryl

A NS16550A-compatible UART implemented in [Veryl](https://veryl-lang.org/) (v0.20.1),
a modern hardware description language that transpiles to SystemVerilog.

## Features

- AXI4-Lite slave interface (5-bit address, 32-bit data)
- 16-entry TX and RX FIFOs (backed by `$std::fifo`)
- 16× oversampling receiver with start-bit detection
- Configurable word length (5–8 bits), parity (none/odd/even/stick), and stop bits (1/2)
- Baud rate generator: 16-bit divisor, `Baud = CLK_FREQ / (divisor × 16)`
- Full register set: RBR, THR, DLL, DLM, IER, IIR, FCR, LCR, MCR, LSR, MSR, SCR
- Interrupt output with four sources: RDA, RX timeout, THRE, LSR errors, MSR change
- RX timeout interrupt (fires after ~4 character times of inactivity)
- Loopback mode (MCR.LOOP): TX→RX, RTS→CTS internally
- Synchronous active-low reset; positive-edge clock

## Quick start

```bash
# Prerequisites: Veryl 0.20.1, Verilator
veryl build          # transpile to SystemVerilog (output in target/)
veryl test           # run all tests with Verilator
veryl test -t uart_16550   # run only the top-level integration tests
```

See `CLAUDE.md` for the full list of build and test commands.

## Module structure

| Source file | Module | Description |
|---|---|---|
| `src/baud_gen.veryl` | `BaudGen` | 16× baud-rate enable from `{DLM, DLL}` divisor |
| `src/uart_tx.veryl` | `UartTx` | TX FIFO (16×8-bit) + shift register |
| `src/uart_rx.veryl` | `UartRx` | 16× oversampling RX with error detection + FIFO |
| `src/uart_regs.veryl` | `UartRegs` | AXI4-Lite slave, register storage and decode |
| `src/uart_16550.veryl` | `Uart16550` | Top level: wires sub-modules, LSR/MSR/IIR, interrupts |

`Uart16550` is the top-level module. The full port list and register map are in `SPEC.md`.

## Ports (summary)

| Signal | Dir | Description |
|---|---|---|
| `i_clk` | in | System clock (positive edge) |
| `i_rst_n` | in | Synchronous active-low reset |
| `i_axi_aw*` / `i_axi_w*` / `o_axi_b*` | — | AXI4-Lite write channel |
| `i_axi_ar*` / `o_axi_r*` | — | AXI4-Lite read channel |
| `i_uart_rxd` / `o_uart_txd` | in/out | Serial data |
| `i_uart_cts` / `o_uart_rts` | in/out | Hardware flow control (active-low RS-232) |
| `o_intr` | out | Interrupt (active-high level, gated by MCR.OUT2) |

Parameter `CLK_FREQ` (default `50_000_000`) must match the actual clock frequency
so the baud rate divisor produces the correct rate.

## Register map (brief)

| Offset | DLAB | Reg | Notes |
|---|---|---|---|
| 0x00 | 0 | RBR / THR | Read pops RX FIFO; write pushes TX FIFO |
| 0x00 | 1 | DLL | Divisor latch low byte |
| 0x04 | 0 | IER | Interrupt enable (bits 3:0) |
| 0x04 | 1 | DLM | Divisor latch high byte |
| 0x08 | — | IIR (R) / FCR (W) | Interrupt ID / FIFO control |
| 0x0C | — | LCR | Line control (word length, parity, stop bits, DLAB) |
| 0x10 | — | MCR | Modem control (RTS, OUT2, loopback) |
| 0x14 | — | LSR | Line status (DR, OE, PE, FE, BI, THRE, TEMT, FIFOERR) |
| 0x18 | — | MSR | Modem status (DCTS, CTS) |
| 0x1C | — | SCR | Scratch register |

Full bit-field descriptions are in `SPEC.md`.

## AXI4-Lite handshake behaviour

- AW and W channels are accepted in the same cycle (each ready depends on the
  other's valid), so a master may present them together or in any order.
- BRESP follows one cycle after acceptance, held until BREADY.
- AR is accepted in one cycle; RDATA/RVALID appear the next cycle.
- All responses are OKAY (`2'b00`).
- Reading RBR pops the RX FIFO; reading LSR clears OE; reading MSR clears DCTS.

## Tests

Each module has a dedicated test file under `test/`. All tests use embedded
SystemVerilog testbenches and the `$tb::clock_gen` infrastructure.

| Test file | Coverage |
|---|---|
| `test/test_baud_gen.veryl` | `o_en` period, divisor change |
| `test/test_uart_tx.veryl` | Bit-exact TX stream, parity, stop bits, FIFO full |
| `test/test_uart_rx.veryl` | Valid frame, framing/parity/overrun errors |
| `test/test_uart_regs.veryl` | AXI handshake, DLAB, FCR pulses, read strobes |
| `test/test_uart_16550.veryl` | Config readback, TX bitstream, loopback, interrupts, multi-byte stream, >16-byte overflow, 115200-baud |

Run all 25 tests with:

```bash
veryl test --sim verilator
```

## Known limitations and future work

See [`FUTURE_WORK.md`](FUTURE_WORK.md) for a detailed list. The main gaps relative
to a full NS16550A are:

1. **Modem inputs absent** — DSR#, RI#, DCD# pins not implemented; MSR[7:5] is
   hard-wired to `101` and delta bits DDSR/TERI/DDCD always read 0.
2. **DTR# and OUT1# outputs absent** — MCR bits 0 and 2 are writable but not
   brought out to physical pins.
3. **FIFOERR approximation** — LSR bit 7 reflects only the RX FIFO head entry,
   not the entire FIFO.
4. **1.5 stop bits not implemented** — WLS=00, STB=1 sends 2 stop bits instead
   of 1.5.
5. **THRE interrupt acknowledge** — IIR read does not clear the THRE interrupt
   (no `r_thre_armed` flag).
6. **Fixed RX timeout** — hardcoded to 640 baud-en ticks (8N1); not adjusted for
   shorter frame formats.
7. **No DMA support** — RXRDY#/TXRDY# pins and FCR.DMASEL are not implemented.
