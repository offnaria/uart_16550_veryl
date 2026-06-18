# Future Work

Known gaps between this implementation and a full NS16550A.
Items are ordered roughly by practical impact.

## 1. Modem-status input pins (DSR#, RI#, DCD#)

The real NS16550A has three active-low modem-input pins that are absent here.

| Pin | MSR bit | Delta bit |
|---|---|---|
| DSR# (Data Set Ready) | 5 DSR | 1 DDSR |
| RI# (Ring Indicator) | 6 RI | 2 TERI |
| DCD# (Data Carrier Detect) | 7 DCD | 3 DDCD |

Currently MSR[7:5] is hard-wired to `101` and MSR[3:1] always reads 0.
To fix: add `i_uart_dsr`, `i_uart_ri`, `i_uart_dcd` input ports; invert them
into MSR[7:5]; add sticky delta bits with clear-on-MSR-read behaviour (same
pattern as the existing DCTS / r_dcts logic).

Loopback mode also needs the missing signal routing: DTR→DSR, OUT1→RI,
OUT2→DCD (only RTS→CTS is currently wired).

## 2. DTR# and OUT1# output pins

MCR bits 0 (DTR) and 2 (OUT1) are writable but not brought out as pins.
The real chip drives active-low `DTR#` and `OUT1#` outputs.
To fix: add `o_uart_dtr` and `o_uart_out1` output ports driven by
`!w_mcr[0]` and `!w_mcr[2]` (held high in loopback mode, same as RTS).

## 3. FIFOERR tracks only the FIFO head, not all entries

LSR bit 7 (FIFOERR) is specified as "any error in RX FIFO" — it should be
set whenever *any* entry in the 16-entry RX FIFO has a PE, FE, or BI error,
not just the current head.

Current implementation: `FIFOERR = w_bi | w_fe | w_pe` (head entry only).

To fix: maintain a saturating error counter (increment on push with errors,
decrement on pop if that entry had errors) or a single sticky bit that is set
on any error push and cleared only when the FIFO becomes empty or is flushed.

## 4. 1.5 stop bits for 5-bit word length

LCR: STB=1 with WLS=00 (5-bit words) should produce 1.5 stop bits (24
baud-en ticks), not 2 full stop bits (32 ticks). The SPEC.md documents this
correctly but `UartTx` always sets `r_stop2 = i_lcr_stb`, which sends 2 stop
bits regardless of word length.

To fix: in `UartTx`, check `i_lcr_stb && (i_lcr_wls == 2'b00)` to enter a
half-stop state that counts to 8 ticks instead of 16 before returning to Idle.

Rarely used in practice (5-bit Baudot/Murray code only).

## 5. THRE interrupt acknowledge on IIR read

The original NS16550A clears the THRE interrupt when IIR is read while it
reports ID=001 (THRE). The interrupt re-asserts only after the next TX FIFO
empty event. Our IIR is purely combinational, so the interrupt stays asserted
continuously while the TX FIFO is empty and ETBEI is set.

Most `8250`-compatible drivers (including Linux) handle this correctly anyway,
but it is a protocol deviation from the datasheet.

To fix: add a registered `r_thre_armed` flag; set it when a THR write drains
the FIFO to empty, clear it when IIR is read with ID=THRE; gate
`w_thre_pending` with `r_thre_armed`.

## 6. RX timeout threshold is fixed at 8N1 timing

The RX timeout counter in `Uart16550` fires after 640 baud-en ticks, which
equals 4 character times for 8N1 frames (10 bits × 16 × 4 = 640). For shorter
frames the threshold is proportionally too long: e.g., 5N1 should use
4 × 7 × 16 = 448 ticks.

To fix: compute the threshold combinationally from LCR:
`threshold = 64 * (7 + {8'b0, w_lcr[1:0]} + {9'b0, w_lcr[3]})`.

## 7. RXRDY# / TXRDY# DMA handshake pins

FCR bit 3 (DMASEL) in the real NS16550A controls the behaviour of `RXRDY#`
and `TXRDY#` output pins used for DMA burst transfers. These pins and the
DMASEL logic are entirely absent. Not needed for interrupt-driven or
memory-mapped use, but required for DMA-mode operation.
