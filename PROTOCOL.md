# Claro Whisper DC — RF protocol

Decoded 2026-08-25 from RTL2832U (rtl_433 25.12) captures. See `captures/BUTTONS.md` for the raw button→code table and `captures/` for the underlying logs.

## Physical layer

| Property     | Value |
|--------------|-------|
| Frequency    | 433.92 MHz |
| Modulation   | OOK, PWM |
| Short pulse  | ~412 µs |
| Long pulse   | ~1192 µs |
| Gap / reset  | ~11.9 ms between bursts |
| Repeats      | 4 identical frames per button press (held keys repeat the same frame) |
| Code type    | **Fixed code — no rolling counter. Fully replayable.** |

rtl_433 flex decoder:

```
-X 'n=claro,m=OOK_PWM,s=412,l=1192,r=11932,g=1172,t=312,y=0'
```

## Frame layout — 33 bits

```
 byte 0     byte 1     byte 2     byte 3     bit
[addr_hi ] [addr_lo ] [command] [checksum] [1]
 \_____ 16-bit address _____/                ^ trailing stop bit (always 1)
```

- **Address (16 bit):** per-remote identifier. Constant across every button of a given remote.
  - Remote A = `0x3123`
  - Remote B = `0x9221`
- **Command (8 bit):** the button. **Identical across remotes** for the same button.
- **Checksum (8 bit):** `checksum = addr_hi XOR addr_lo XOR command`.
  Equivalently, `addr_hi XOR addr_lo XOR command XOR checksum == 0` (byte-wide XOR parity).
- **Stop bit:** a single `1` after byte 3 (the 33rd bit).

Verified against all 19 remote-A codes and all 5 remote-B sample codes — zero exceptions.

## Command byte table

| Button        | cmd  | Button        | cmd  |
|---------------|------|---------------|------|
| Speed 1       | `fb` | CCT Natural   | `fd` |
| Speed 2       | `f5` | CCT Cool      | `f6` |
| Speed 3       | `f7` | Dim −         | `f2` |
| Speed 4       | `f3` | Dim +         | `f0` |
| Speed 5       | `f9` | Timer 2H      | `ef` |
| Speed 6       | `fa` | Timer 4H      | `ee` |
| FAN/OFF       | `f4` | Timer 8H      | `ed` |
| Power         | `fe` | Breeze (≋)    | `ec` |
| Light on/off  | `fc` | F/R (reverse) | `ea` |
| CCT Warm      | `f8` |               |      |

(Fan/light commands fall in the `0xf_` range, timer/breeze/reverse in the `0xe_` range — an observation, not needed for control.)

## Per-fan addressing — verdict

**Yes, the two fans are individually addressable.** Each remote carries a distinct 16-bit address; the fan receiver is paired to its remote's address (learn-on-power-up, per the manual). Transmitting with address `0x3123` drives fan A only; `0x9221` drives fan B only.

## Synthesising a full command set for any fan

Because the command byte is address-independent and the checksum is a pure function of address+command, the complete 19-button set for **any** address can be generated without capturing that remote:

```
frame(addr, cmd):
    chk = (addr >> 8) ^ (addr & 0xff) ^ cmd
    payload32 = (addr << 16) | (cmd << 8) | chk
    # transmit payload32 as 32 OOK-PWM bits, then a single '1' stop bit
```

This was validated by synthesising remote B's speed1/2/3, light, and F/R from remote A's command bytes + address `0x9221` — all five matched the captured B codes exactly. Only the 16-bit address of each physical fan must be known (read from any single button capture of its remote).

## Implications for later tickets

- **Transmit (ESPHome):** send raw OOK-PWM (412/1192 µs, ~4 repeats, 11.9 ms gap) via CC1101. A small template that composes `[addr][cmd][chk][1]` is cleaner than 38 hardcoded raw frames (19 buttons × 2 fans). Fan B's full set is derived, not captured.
- **State:** fixed-code and one-way; HA state must be optimistic/assumed (see map's "Not yet specified").
