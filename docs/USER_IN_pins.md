<!-- [MiSTer-DB9 BEGIN] - DB9/SNAC8 support -->
# USER_IN Pin Mapping

## Overview

The MENU core uses the MiSTer USB port `USER_IN` pins (`USR2-8`) to support DB9MD, DB15, and Saturn joystick interfaces. These pins are multiplexed between the three joystick protocols, with autodetection handling protocol switching.

**File**: `menu.sv`
**Interface**: `input [7:0] USER_IN`

## Pin Assignment

| USER_IN | DB9MD | DB15 | Saturn | MT32pi | Description |
|---------|-------|------|--------|--------|-------------|
| [0]     | no    | no   | yes    | yes    | Saturn D1 / MT32pi I2C SDA |
| [1]     | yes   | no   | yes    | no     | DB9MD data / Saturn D0 |
| [2]     | yes   | yes  | no     | no     | DB9MD/DB15 disable signal (Saturn uses IO[2] as SPLIT output) |
| [3]     | yes   | yes  | yes    | no     | DB9MD/DB15 disable signal / Saturn D3 |
| [4]     | no    | no   | out    | yes    | Saturn S0 push-pull / MT32pi I2S BCLK |
| [5]     | yes   | yes  | yes    | yes    | DB9MD/DB15 data / Saturn D2 / MT32pi I2S data |
| [6]     | yes   | yes  | out    | yes    | DB9MD status / DB15 disable / Saturn S1 push-pull / MT32pi I2S WS |
| [7]     | yes   | no   | no     | yes    | DB9MD status / MT32pi I2S WS |

## Saturn Protocol

### Implementation
**Module**: `joydb9saturn.v`
**Clock**: `CLK_50M`
**Data Bus**: `JOY_SATURN_IN = {USER_IN[3], USER_IN[5], USER_IN[0], USER_IN[1]}` = {D3, D2, D1, D0}
**Select Lines**: `USER_IO[4]` = S0 (push-pull), `USER_IO[6]` = S1 (push-pull)
**2P Splitter**: `USER_IO[2]` = SPLIT (push-pull, selects P1/P2 side of 74HC157D mux)

### Autodetection
Saturn is detected using a settle → probe cycle:

1. **Settle** (~10ms): all pins float for IO recovery from DB15/DB9MD overdrive.
2. **Probe** (~5ms): Saturn module scans with push-pull S0/S1/SPLIT. Detection checked every `clk_sys` cycle.
3. After 2 probe cycles without detection → timeout → idle (DB15/DB9MD operate normally).
4. In idle: DB15 captures data for ~10ms. If `db15_idle=1` → re-enter settle.
5. At boot: `saturn_probe=1` from init → module scans immediately with real inputs (no settle needed).

Detection uses the fixed protocol signature D1=D0=0 in the {S0=1,S1=1} phase (Saturn SMPC PAD_DIGITAL ID). A 4-bit shift-register debounce requires 1 hit to connect and 4 consecutive misses to disconnect.

### Saturn Data Leak Prevention
During all Saturn phases (`saturn_any=1`), `JOY_DATA` and `JOY_MDIN` are gated to `'1` to prevent the Saturn pad's D2 output (IO[5]) from leaking into the DB15 module and causing false `db15_idle=0`.

### Audio Noise Prevention
During Saturn probe, S0/S1 toggling is misinterpreted by the I2S decoder as `i2s_bclk`. The I2S registers (`mt32_i2s_l/r`) are cleared during `saturn_any` and audio outputs are muted.

## DB9MD Protocol (Mega Drive/Genesis)

### Implementation
**Module**: `joy_db9md.v`
**Clock**: `CLK_50M`
**Data Bus**: `JOY_MDIN = {USER_IN[6], USER_IN[3], USER_IN[5], USER_IN[7], USER_IN[1], USER_IN[2]}`

### Auto-Detection
- `USER_IN[7]` LOW triggers DB9MD mode (`db9md_ena=1`).
- DB9MD detection fires when `~saturn_mode` (IO[7] readable when not in Saturn probe).
- Once active, DB9MD keeps mode until both decoded ports stay idle for ~125ms.
- When idle timeout expires, `db9md_ena` and `db15_disable` are cleared so DB15 can take over.

## DB15 Protocol (Neo-Geo/Supergun)

### Implementation
**Module**: `joy_db15.v`
**Clock**: `CLK_50M`
**Data Bus**: `JOY_DATA = USER_IN[5]`
**Controls**: `JOY_CLK` (IO[1]), `JOY_LOAD` (IO[0])

### Auto-Disable Logic
```verilog
if(~saturn_any & (~USER_IN[6] || ~USER_IN[2] || ~USER_IN[3])) db15_disable <= 1'b1;
```
Gated on `~saturn_any` to prevent false triggers from Saturn push-pull pins.

## State Machine

### States
- **saturn_active**: Saturn pad detected, running. `saturn_mode=1`.
- **saturn_probe**: Saturn module scanning with push-pull S0/S1/SPLIT. `saturn_mode=1`.
- **saturn_settle**: All pins float for IO recovery. `saturn_mode=0`, `saturn_any=1`.
- **Idle**: DB15/DB9MD operate normally. `saturn_mode=0`, `saturn_any=0`.

### Transitions
```
idle(10ms) → settle(10ms) → probe(5ms) → settle(10ms) → probe(5ms) → idle (timeout)
                                        → saturn_active (if detected)
saturn_active → idle (on disconnect, sets db9_1p_ena=1 and clears db15_disable)
```

### USER_MODE / USER_PP
| State | USER_MODE | USER_PP | Description |
|-------|-----------|---------|-------------|
| Idle (DB15) | 2'b01 | 8'b00000000 | DB15 CLK/LOAD push-pull on IO[0]/IO[1] |
| Idle (DB9MD) | 2'b10 | 8'b00000000 | DB9MD MDSEL push-pull on IO[0] |
| Saturn probe/active | 2'b00 | 8'b01010100 | Push-pull S0(IO[4]), S1(IO[6]), SPLIT(IO[2]) |
| Settle/gap | 2'b00 | 8'b00000000 | All float for IO recovery |

## References

- `menu.sv`: joystick detection and muxing
- `joydb9md.v`: DB9MD protocol implementation
- `joydb15.v`: DB15 protocol implementation
- `joydb9saturn.v`: Saturn protocol implementation with 2P splitter support
- `sys/mt32pi.sv`: MT32pi `USER_IN` usage
<!-- [MiSTer-DB9 END] -->
