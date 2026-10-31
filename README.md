# RISC-V Based SoC with PWM, Watchdog Timer and Seven-Segment Display Controllers

A complete RISC-V System-on-Chip integrating a VeeR EL2 RISC-V processor core with
AXI interconnect and three custom project IPs: a 4-channel **PWM controller**, a
**Watchdog Timer**, and a 4-digit **Seven-Segment Display controller**.

Verified end-to-end in **Synopsys VCS** and **Verdi** (26/26 tests pass).

---

## Architecture Overview

```
                    +---------------------------------------------------+
                    |                  soc_top                           |
                    |                                                   |
  +----------+      |  +-----------+   AXI 64-to-32   +----------------+  |
  |  VeeR    |  AXI4 |  | Width Adp |  (LSU/IFU/SB)  |  5x20 AXI     |  |
  |  EL2     |------>|  +-----------+---------------->|  Crossbar     |  |
  | RV32IMC  | 64-bit|                               |  Interconnect |  |
  |  Core    |      |  +------------+                +-------+-------+  |
  +----------+      |  |  axi_ram   |  IMEM/DMEM   |    |   |   |   |  |
                    |  +------------+               v    v   v   v   v  |
                    |                          +----+--+--+---+---+--+ |
                    |  Peripherals (AXI4-Lite):  |PWM|SSD|TMR|GPIO|WDT| |
                    |  + SPI + UART              +---+---+---+---+---+  |
                    +---------------------------------------------------+
```

### SoC Components

| # | Component | Base Address | Bus | Description |
|---|-----------|-------------|-----|-------------|
| M00 | AXI UART | 0x4000_6000 | AXI4 | Serial debug (115200 baud) |
| M01 | IMEM | 0x1000_0000 | AXI4 | 64 KB instruction memory |
| M02 | DMEM | 0x1002_0000 | AXI4 | 64 KB data memory |
| M03 | (reserved) | 0x4000_0000 | — | Stubbed (DECERR) |
| **M04** | **PWM Controller** | **0x4000_2000** | AXI4-Lite | **4-channel PWM — Project IP** |
| **M05** | **Seven-Segment Display** | **0x4000_4000** | AXI4-Lite | **4-digit mux'd SSD — Project IP** |
| M06 | Timer | 0x4000_8000 | AXI4-Lite | Acquisition timer w/ sample_tick |
| M07 | GPIO | 0x4000_A000 | AXI4-Lite | 8-bit I/O with edge IRQ |
| M08 | SPI Master | 0x4000_C000 | AXI4-Lite | Auxiliary SPI peripheral |
| **M09** | **Watchdog Timer** | **0x4000_E000** | AXI4-Lite | **Safety monitor w/ NMI & reset trip — Project IP** |

---

## Project IPs (Your Custom Hardware)

### 1. PWM Controller (`rtl/pwm/pwm.sv`)
4-channel pulse-width modulation generator with shared period register.

| Offset | Register | Description |
|--------|----------|-------------|
| 0x00 | PWM_CTRL | [3:0] per-channel enable |
| 0x04 | PWM_PERIOD | [15:0] common period (clk cycles) |
| 0x08 | PWM_DUTY0 | [15:0] channel-0 duty |
| 0x0C | PWM_DUTY1 | [15:0] channel-1 duty |
| 0x10 | PWM_DUTY2 | [15:0] channel-2 duty |
| 0x14 | PWM_DUTY3 | [15:0] channel-3 duty |
| 0x18 | PWM_STATUS | [3:0] running (RO) |
| 0x1C | PWM_IRQ_CTRL | [0] period-rollover IRQ enable |

Outputs: `pwm_out[3:0]`

### 2. Seven-Segment Display Controller (`rtl/seven_segment/seven_segment.sv`)
4-digit multiplexed 7-segment display with built-in hex decoder.

| Offset | Register | Description |
|--------|----------|-------------|
| 0x00 | SSD_CTRL | [0] EN, [1] MODE (0=hex, 1=raw) |
| 0x04 | SSD_DATA | [15:0] display value (nibble per digit) |
| 0x08 | SSD_BRIGHT | [7:0] refresh prescaler |
| 0x0C | SSD_STATUS | [0] running (RO) |

Outputs: `ssd_seg[7:0]` (active-low cathodes), `ssd_digit_sel[3:0]` (active-low anodes)

### 3. Watchdog Timer (`rtl/watchdog/watchdog.sv`)
Safety monitor and fault recovery timer with two-stage timeout (NMI warning and hard system reset).

| Offset | Register | Description |
|--------|----------|-------------|
| 0x00 | WDT_CTRL | [0] ENABLE, [1] NMI_EN |
| 0x04 | WDT_RELOAD | [31:0] reload duration in clock cycles |
| 0x08 | WDT_COUNT | [31:0] instantaneous countdown value (RO) |
| 0x0C | WDT_KICK | Write `0x55AA_1234` to service/kick counter |
| 0x10 | WDT_STATUS | [0] TRIPPED sticky flag (W1C) |

Outputs: `wdt_nmi`, `wdt_reset`

---

## How to Run (VCS + Verdi)

### Prerequisites
Set up the Synopsys tool environment:
```bash
export VCS_HOME=/home/student/snps_tools_target/vcs/U-2023.03
export VERDI_HOME=/home/student/snps_tools_target/verdi/U-2023.03-SP1
export SNPSLMD_LICENSE_FILE=27021@14.139.1.126
export PATH=$VCS_HOME/bin:$VERDI_HOME/bin:$PATH
```

### Quick Start
```bash
# Compile and simulate the full SoC
make soc

# Open waveforms in Verdi
make soc-verdi
```

### Manual (without Make)
```bash
cd run
./run_vcs_soc.sh           # compile + simulate
./run_vcs_soc.sh verdi     # compile + simulate + open Verdi
```

---

## File Structure (Files Modified/Created for This Project)

```
rtl/pwm/pwm.sv                         # NEW — 4-channel PWM controller (AXI4-Lite)
rtl/seven_segment/seven_segment.sv     # NEW — 4-digit SSD controller (AXI4-Lite)
rtl/watchdog/watchdog.sv               # NEW — Watchdog Timer (AXI4-Lite)
rtl/soc_top.sv                         # MODIFIED — SoC top-level
tb/tb_soc_top.sv                        # MODIFIED — self-checking testbench
tb/adc_model.sv                         # REUSED — behavioral peripheral model
run/filelist_soc.f                      # MODIFIED — VCS file list
run/run_vcs_soc.sh                      # NEW — VCS+Verdi run script
Makefile                               # MODIFIED — project name & help
README.md                              # NEW — this file
```

### Shared Infrastructure (unchanged, from open-source reference)
- `rtl/veer_el2/` — Western Digital VeeR EL2 RISC-V core (RV32IMC)
- `rtl/interconnect/` — 5x20 AXI4 crossbar interconnect
- `rtl/axi_uart/` — AXI UART 16550 IP
- `lib/axi4_to_axil_lite.sv` — AXI4-Full to AXI4-Lite bridge
- `lib/axi_ram.sv` — AXI4 on-chip memory
- `rtl/interconnect/axi_adapter_64_to_32.sv` — bus width adapter

---

## Memory Map Summary

```
0x1000_0000  IMEM        (64 KB)   Instruction memory
0x1002_0000  DMEM        (64 KB)   Data memory
0x4000_2000  PWM         (4 KB)   4-channel PWM controller     [Project IP]
0x4000_4000  SSD         (4 KB)   Seven-segment display        [Project IP]
0x4000_6000  UART        (4 KB)   Serial debug
0x4000_8000  TIMER       (4 KB)   Acquisition timer
0x4000_A000  GPIO        (4 KB)   General-purpose I/O
0x4000_C000  SPI         (4 KB)   SPI master
0x4000_E000  WATCHDOG    (4 KB)   Safety monitor Watchdog      [Project IP]
```

---

## Test Results

All 26 assertions pass in VCS simulation:

```
Total Assertions Checked : 26
Total Passed             : 26
Total Failed             : 0
Total Instructions Retired: 3671
>>> ALL RISC-V SoC TESTS PASSED! <<<
```

Tests covered:
- Power-on reset & initialization
- VeeR EL2 instruction fetch/retire from on-chip IMEM
- UART serial transmission (Tx 'A')
- GPIO software control (0xA5)
- **PWM** — 4 channels configured, outputs toggling
- **Seven-Segment Display** — data, multiplexing, segment drive
- **Watchdog Timer** — safety monitor w/ NMI & reset trip, timeout countdown & kick reload
- DMEM read/write
- UART RX interrupt
- SPI peripheral communication
- PIC interrupt routing
