# UART 16550A — Design Specification

## Parameters

| Name | Type | Default | Description |
|---|---|---|---|
| `CLK_FREQ` | `u32` | `50_000_000` | System clock frequency in Hz |

## Ports

| Signal | Dir | Width | Description |
|---|---|---|---|
| `i_clk` | in | clock | System clock (positive edge) |
| `i_rst_n` | in | reset | Synchronous active-low reset |
| `i_axi_awvalid` | in | 1 | AXI write address valid |
| `o_axi_awready` | out | 1 | AXI write address ready |
| `i_axi_awaddr` | in | 32 | AXI write address |
| `i_axi_awprot` | in | 3 | AXI write protection (ignored) |
| `i_axi_wvalid` | in | 1 | AXI write data valid |
| `o_axi_wready` | out | 1 | AXI write data ready |
| `i_axi_wdata` | in | 32 | AXI write data |
| `i_axi_wstrb` | in | 4 | AXI write strobe (byte enables) |
| `o_axi_bvalid` | out | 1 | AXI write response valid |
| `i_axi_bready` | in | 1 | AXI write response ready |
| `o_axi_bresp` | out | 2 | AXI write response (always OKAY=00) |
| `i_axi_arvalid` | in | 1 | AXI read address valid |
| `o_axi_arready` | out | 1 | AXI read address ready |
| `i_axi_araddr` | in | 32 | AXI read address |
| `i_axi_arprot` | in | 3 | AXI read protection (ignored) |
| `o_axi_rvalid` | out | 1 | AXI read data valid |
| `i_axi_rready` | in | 1 | AXI read data ready |
| `o_axi_rdata` | out | 32 | AXI read data |
| `o_axi_rresp` | out | 2 | AXI read response (always OKAY=00) |
| `i_uart_rxd` | in | 1 | Serial receive |
| `o_uart_txd` | out | 1 | Serial transmit |
| `i_uart_cts` | in | 1 | Clear To Send (active-low per RS-232) |
| `o_uart_rts` | out | 1 | Request To Send (active-low per RS-232) |
| `o_intr` | out | 1 | Interrupt output, active-high level |

## Register Map

`reg_shift=2` (word-aligned, offsets in bytes). All registers are 8-bit; upper bits of the 32-bit AXI word are ignored on write and read as zero.

| Offset | DLAB | R/W | Reg | Description |
|---|---|---|---|---|
| 0x00 | 0 | R | RBR | Receiver Buffer Register |
| 0x00 | 0 | W | THR | Transmitter Holding Register |
| 0x00 | 1 | R/W | DLL | Divisor Latch Low byte |
| 0x04 | 0 | R/W | IER | Interrupt Enable Register |
| 0x04 | 1 | R/W | DLM | Divisor Latch High byte |
| 0x08 | — | R | IIR | Interrupt Identification Register |
| 0x08 | — | W | FCR | FIFO Control Register |
| 0x0C | — | R/W | LCR | Line Control Register |
| 0x10 | — | R/W | MCR | Modem Control Register |
| 0x14 | — | R | LSR | Line Status Register |
| 0x18 | — | R | MSR | Modem Status Register |
| 0x1C | — | R/W | SCR | Scratch Register |

### IER (0x04, DLAB=0)
| Bit | Name | Description |
|---|---|---|
| 0 | ERBFI | Enable Received Data Available interrupt |
| 1 | ETBEI | Enable TX Holding Register Empty interrupt |
| 2 | ELSI | Enable Receiver Line Status interrupt |
| 3 | EDSSI | Enable Modem Status interrupt |
| 7:4 | — | Reserved, read 0 |

### IIR (0x08, read)
| Bit | Name | Description |
|---|---|---|
| 0 | nIP | 0 = interrupt pending, 1 = no interrupt |
| 3:1 | ID | 011=LSR, 010=RDA, 110=RX timeout, 001=THRE, 000=MSR |
| 5:4 | — | Reserved, read 0 |
| 7:6 | FIFOEN | Always 11 (FIFOs always enabled) |

Interrupt priority (highest→lowest): LSR errors → RDA/timeout → THRE → MSR.

### FCR (0x08, write)
| Bit | Name | Description |
|---|---|---|
| 0 | FIFOEN | FIFO enable — ignored, FIFOs always on |
| 1 | RXRST | Reset RX FIFO (self-clearing) |
| 2 | TXRST | Reset TX FIFO (self-clearing) |
| 3 | DMASEL | DMA mode — ignored |
| 5:4 | — | Reserved |
| 7:6 | RXTRIG | RX FIFO trigger level: 00=1, 01=4, 10=8, 11=14 |

### LCR (0x0C)
| Bit | Name | Description |
|---|---|---|
| 1:0 | WLS | Word length: 00=5, 01=6, 10=7, 11=8 bits |
| 2 | STB | Stop bits: 0=1, 1=1.5(WLS=00)/2 |
| 3 | PEN | Parity enable |
| 4 | EPS | Even parity select |
| 5 | SP | Stick parity |
| 6 | BC | Break control |
| 7 | DLAB | Divisor Latch Access Bit |

### MCR (0x10)
| Bit | Name | Description |
|---|---|---|
| 0 | DTR | Data Terminal Ready — writable, no physical pin |
| 1 | RTS | Request To Send → drives `o_uart_rts` (active-low) |
| 2 | OUT1 | Output 1 — writable, no physical pin |
| 3 | OUT2 | Output 2 — gates `o_intr` (must be 1 for interrupts) |
| 4 | LOOP | Loopback mode |
| 7:5 | — | Reserved, read 0 |

### LSR (0x14, read-only)
| Bit | Name | Description |
|---|---|---|
| 0 | DR | Data Ready (RX FIFO not empty) |
| 1 | OE | Overrun Error (cleared on read) |
| 2 | PE | Parity Error at head of RX FIFO |
| 3 | FE | Framing Error at head of RX FIFO |
| 4 | BI | Break Interrupt at head of RX FIFO |
| 5 | THRE | TX Holding Register Empty (TX FIFO not full) |
| 6 | TEMT | Transmitter Empty (TX FIFO empty AND shift reg idle) |
| 7 | FIFOERR | Any error in RX FIFO |

### MSR (0x18, read-only)
| Bit | Name | Description |
|---|---|---|
| 0 | DCTS | Delta CTS — set when CTS changes, cleared on read |
| 3:1 | — | DDSR/TERI/DDCD — read 0 |
| 4 | CTS | Current state of `i_uart_cts` (active-low inverted to active-high) |
| 7:5 | DSR/RI/DCD | Tied: DSR=1, RI=0, DCD=1 |

## Architecture

### Source files
| File | Module | Description |
|---|---|---|
| `src/fifo16.veryl` | `Fifo16` | 16-entry FWFT FIFO, parameterized WIDTH |
| `src/baud_gen.veryl` | `BaudGen` | 16x baud clock enable from CLK_FREQ and DL divisor |
| `src/uart_tx.veryl` | `UartTx` | TX FIFO + shift register |
| `src/uart_rx.veryl` | `UartRx` | RX shift register + FIFO, 16x oversampling |
| `src/uart_16550.veryl` | `Uart16550` | Top-level: AXI4-Lite, registers, interrupt, loopback |

### Baud rate generation
- 16-bit divisor from {DLM, DLL}
- Baud = CLK_FREQ / (divisor × 16)
- Counter counts from divisor down to 1, generates 1-cycle enable at each rollover
- TX and RX both use this 16x enable

### TX path
THR write → TX FIFO (8-bit, 16 entries) → shift register → `o_uart_txd`  
Shift register loads next byte from FIFO when it becomes idle.

### RX path
`i_uart_rxd` → 16x oversampler (samples at tick 8 of 16) → shift register → RX FIFO (11-bit, 16 entries)  
Each RX FIFO entry: `{BI, FE, PE, data[7:0]}`  
OE (overrun) is flagged when RX FIFO is full and a new byte arrives.

### AXI4-Lite handshake
- AW and W channels accepted simultaneously in one cycle
- BRESP asserted next cycle, stays until BREADY
- AR accepted in one cycle, RDATA/RVALID asserted next cycle
- All responses OKAY (2'b00)
- RBR read advances RX FIFO pointer; reading LSR clears OE; reading MSR clears DCTS

### Interrupt logic
`o_intr = MCR.OUT2 AND (any enabled pending interrupt)`

Priority encoder (highest wins for IIR):
1. LSR: OE | PE | FE | BI and ELSI set
2. RDA: RX FIFO count ≥ trigger level and ERBFI set
3. RX timeout: RX FIFO not empty, no read for 4 char times, ERBFI set
4. THRE: TX FIFO empty and ETBEI set
5. MSR: DCTS set and EDSSI set

### Loopback (MCR.LOOP=1)
- TX shift output → RX input (internally)
- RTS → CTS (internally)
- `o_uart_txd` held high, `o_uart_rts` held high (inactive)
