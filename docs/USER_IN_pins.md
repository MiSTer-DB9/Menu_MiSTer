<!-- [MiSTer-DB9 BEGIN] - DB9/SNAC8 support -->
# USER_IN Pin Mapping

## Overview

The MENU core uses the MiSTer USB port `USER_IN` pins (`USR2-8`) to support DB9 and DB15 joystick interfaces. These pins are multiplexed between the two joystick protocols, with DB9MD taking priority when detected.

**File**: `menu.sv`
**Interface**: `input [7:0] USER_IN`

## Pin Assignment

| USER_IN | DB9MD | DB15 | MT32pi | Description |
|---------|-------|------|--------|-------------|
| [0]     | no    | no   | yes    | MT32pi I2S BCLK |
| [1]     | yes   | no   | no     | DB9MD controller data pin |
| [2]     | yes   | yes  | no     | DB9MD/DB15 disable signal |
| [3]     | yes   | yes  | no     | DB9MD/DB15 disable signal |
| [4]     | no    | no   | yes    | MT32pi I2S data (crossed) |
| [5]     | yes   | yes  | yes    | DB9MD/DB15 data / MT32pi I2S data / MIDI RX |
| [6]     | yes   | yes  | yes    | DB9MD status / DB15 disable / MT32pi I2S WS |
| [7]     | yes   | no   | yes    | DB9MD status / MT32pi I2S WS |

## DB9MD Protocol (Mega Drive/Genesis)

### Implementation
**Module**: `joy_db9md.v`
**Clock**: `CLK_50M`
**Data Bus**: `JOY_MDIN = {USER_IN[6], USER_IN[3], USER_IN[5], USER_IN[7], USER_IN[1], USER_IN[2]}`

### Reading Protocol
The DB9MD controller uses a timing-based protocol similar to the original Genesis controller. The USB `USER_IN` pins are multiplexed but read through a state machine:

1. `joy_split`: toggles between Player 1 and Player 2 sampling.
2. `joy_mdsel`: selects the current read stage.
3. State machine sequence:
   - `state 0-1`: set `joyMDsel`
   - `state 2`: read `CBUDLR`
   - `state 3`: read `Start/A`, detect controller type
   - `state 4`: reset `joyMDsel`
   - `state 5`: detect 6-button controller
   - `state 6`: read `Mode/X/Y/Z`

### Auto-Detection
```verilog
wire db9_status = db9md_ena ? 1'b1 : USER_IN[7];
wire db9_idle = ~(|JOYDB9MD_1[11:0] | |JOYDB9MD_2[11:0]);

if(db9md_ena && db9_idle) begin
    if(db9_idle_cnt < 24'd12499999) db9_idle_cnt <= db9_idle_cnt + 1'd1;
    else begin
        db9md_ena <= 1'b0;
        db15_disable <= 1'b0;
    end
end else db9_idle_cnt <= 24'd0;

if(~db9md_ena & ~db9_status) db9md_ena <= 1'b1;
```

- `USER_IN[7]` is checked to enter DB9MD mode.
- Once DB9MD is active, the design keeps that mode until both decoded DB9MD ports stay idle for about 125 ms.
- When that idle timeout expires, `db9md_ena` and the latched DB15 disable are cleared so DB15 can take over again without a reboot.
- This means DB9MD to DB15 switching depends on the DB9MD side going electrically idle.

## DB15 Protocol (Neo-Geo/Supergun)

### Implementation
**Module**: `joy_db15.v`
**Clock**: `CLK_50M`
**Data Bus**: `JOY_DATA = USER_IN[5]`
**Controls**: `JOY_CLK`, `JOY_LOAD`

### Reading Protocol
The DB15 controller uses a SPI-style timing protocol:

1. `JOY_CLK`: clock signal for data transfer.
2. `JOY_DATA`: serial data input from `USER_IN[5]`.
3. `JOY_LOAD`: load signal to capture data.

### Auto-Disable Logic
```verilog
if(~USER_IN[6] || ~USER_IN[2] || ~USER_IN[3]) db15_disable <= 1'b1;
```

- `db15_disable` is a latch, not a live combinational gate.
- It is asserted when DB9-like activity is seen on `USER_IN[6]`, `USER_IN[2]`, or `USER_IN[3]`.
- It is cleared only by the DB9MD idle timeout path shown above.

## Status Flags

### JOY_FLAG
Three-bit status flag indicating which joystick protocol is currently driven on USER I/O:

- `3'b100`: DB9MD protocol active
- `3'b010`: DB15 protocol active

### USER_MODE
Two-bit output indicating the active joystick protocol:

- `01`: DB15 protocol selected
- `10`: DB9MD protocol selected

## Interoperability

### Simultaneous Connection
- DB9MD and DB15 share USER I/O resources; only one protocol is active at a time.
- DB9MD takes priority when its detect logic sees a valid DB9MD presence.
- Switching from DB9MD to DB15 without reboot depends on the DB9MD decoder becoming idle for about 125 ms.

### MT32pi Interoperability
- MT32pi uses some of the same `USER_IN` pins for audio transport.
- The forked menu core includes the DB9/DB15 multiplexing needed for coexistence.

## References

- `menu.sv`: joystick detection and muxing, around lines 184-240
- `joydb9md.v`: DB9MD protocol implementation
- `joydb15.v`: DB15 protocol implementation
- `sys/mt32pi.sv`: MT32pi `USER_IN` usage
- `README DB9MD Support.md`: user-facing overview
<!-- [MiSTer-DB9 END] -->
