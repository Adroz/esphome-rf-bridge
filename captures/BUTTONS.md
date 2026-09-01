# Claro Whisper DC — captured RF button codes

- **Frequency:** 433.92 MHz, OOK PWM (confirmed on air, SNR ~21 dB)
- **rtl_433 flex decoder:** `-X 'n=claro,m=OOK_PWM,s=412,l=1192,r=11932,g=1172,t=312,y=0'`
- **Code length:** 33 bits, transmitted as 9 hex nibbles. Fixed code (identical on every press) → replayable, no rolling counter.
- **Repeat flag:** repeats within a burst sometimes decode with leading nibble `b` instead of `3`/`9` (e.g. `b123fbe98`). The canonical first transmission uses the `3`/`9` form; the `b` variant is the same payload with a repeat/preamble bit set.
- Raw `.cu8` captures live under `captures/raw-remoteA/` and `captures/raw-remoteB/`; per-button rtl_433 JSON logs are the `remoteA-*.json` / `remoteB-*.json` files.

## Remote A (full — address prefix `3123f`)

| Button        | 33-bit code   |
|---------------|---------------|
| Speed 1       | `3123fbe98`   |
| Speed 2       | `3123f5e78`   |
| Speed 3       | `3123f7e58`   |
| Speed 4       | `3123f3e18`   |
| Speed 5       | `3123f9eb8`   |
| Speed 6       | `3123fae88`   |
| FAN/OFF       | `3123f4e68`   |
| Power (red)   | `3123feec8`   |
| Light on/off  | `3123fcee8`   |
| CCT Warm      | `3123f8ea8`   |
| CCT Natural   | `3123fdef8`   |
| CCT Cool      | `3123f6e48`   |
| Dim −         | `3123f2e08`   |
| Dim +         | `3123f0e28`   |
| Timer 2H      | `3123effd8`   |
| Timer 4H      | `3123eefc8`   |
| Timer 8H      | `3123edff8`   |
| Breeze (≋)    | `3123ecfe8`   |
| F/R (reverse) | `3123eaf88`   |

Notes: Dim − and Dim + repeat the same code while held (hold = repeated frames, not a distinct code).

## Remote B (sample — address prefix `9221f`)

| Button       | 33-bit code   |
|--------------|---------------|
| Speed 1      | `9221fb488`   |
| Speed 2      | `9221f5468`   |
| Speed 3      | `9221f7448`   |
| Light on/off | `9221fc4f8`   |
| F/R          | `9221ea598`   |

## Structure (preliminary — full analysis in PROTOCOL.md via decode ticket)

Aligning A vs B for the same button:

```
          n: 1 2 3 4 5 6 7 8 9
A speed1:    3 1 2 3 f b e 9 8
B speed1:    9 2 2 1 f b 4 8 8
             \___addr__/ \cmd/ \_?_/
```

- **Nibbles 1–4:** per-remote address (`3123` / `9221`), constant across all buttons of one remote.
- **Nibbles 5–6:** button command — **identical across both remotes** for the same button.
- **Nibble 7:** constant per remote (`e` for A, `4`/`5` for B) → part of address/parity.
- **Nibbles 8–9:** vary per button and per remote → checksum over address+command (to be confirmed).

**Checksum (verified against captures):** reading each code as bytes `[id_hi][id_lo][cmd][chk][pad]`, remote A satisfies `chk = cmd XOR 0x12` for **every** button (e.g. `0xfb^0xe9=0x12`, `0xf5^0xe7=0x12`, `0xfc^0xee=0x12`). This does **not** hold for remote B (`0xfb^0x48=0xb3`), so `chk` is a checksum over **address+command**, not a fixed `cmd^0x12` — the `0x12` is remote A's address-derived constant. The decode ticket must model the checksum as address-dependent.

**Implication:** the two fans are individually addressable, and remote B's full 19-command set should be synthesisable from remote A's commands + remote B's address once the address→checksum relationship is cracked. A full remote-B capture is therefore optional, pending the decode ticket.
