# ESPHome control for Claro Whisper DC fans

Drives both fans from Home Assistant via an ESP32-C3 SuperMini + CC1101 (433.92 MHz),
transmitting the decoded protocol (see [`../PROTOCOL.md`](../PROTOCOL.md)) and listening
so HA state follows the physical remotes.

## Status

- **Schema: validated** with `esphome config` (ESPHome 2026.8.1).
- **Firmware compile:** see the repo commit message / CI — run `esphome compile claro-fans.yaml`.
- **On-hardware behaviour: NOT yet verified** — pending the CC1101 module. The two lambdas
  (`send_claro` transmit composer, `on_raw` receive decoder) and the `remote_receiver`
  timing thresholds must be checked against a real radio. Treat pin numbers and the RX
  decoder as first-draft until confirmed.

## Wiring — CC1101 → ESP32-C3 SuperMini

| CC1101 | ESP32-C3 | Notes |
|--------|----------|-------|
| VCC    | **3V3**  | 3.3 V ONLY — 5 V destroys the CC1101 |
| GND    | GND      | |
| SCK    | GPIO4    | SPI clock |
| MISO   | GPIO5    | SPI MISO |
| MOSI   | GPIO6    | SPI MOSI |
| CSN    | GPIO7    | SPI chip select |
| GDO0   | GPIO3    | data OUT (transmit) |
| GDO2   | GPIO10   | data IN (receive) |

Dual-pin wiring (GDO0 for TX, GDO2 for RX) is used deliberately: it gives simultaneous
transmit + receive, and by leaving `gdo0_pin` off the `cc1101:` block it dodges ESPHome
[issue #16876](https://github.com/esphome/esphome/issues/16876) (TX emits no RF when
`gdo0_pin` is set on ESP32).

Use the SMA-antenna CC1101 variant with a 433 MHz whip — antenna match, not TX power,
decides through-wall range (one plaster wall ≈ 1–5 dB loss).

## Flashing

1. Put a `secrets.yaml` next to `claro-fans.yaml` (gitignored):
   ```yaml
   wifi_ssid: "YourSSID"
   wifi_password: "YourPassword"
   ```
2. In the HA ESPHome add-on: add this YAML, or from a workstation:
   `esphome run claro-fans.yaml` (USB for first flash, OTA after).

## Verification checklist (when the CC1101 arrives)

Work top-down; each step gates the next.

1. **Radio comes up:** boot, confirm no SPI errors in logs.
2. **TX reaches the fan:** call `fan_a` speed 1. Fan A should respond. If not:
   - confirm 3V3 power and GDO0 = GPIO3 wiring;
   - check the #16876 workaround (no `gdo0_pin` on the `cc1101:` block);
   - temporarily raise `output_power`, verify antenna is 433 MHz.
3. **Per-fan isolation:** `fan_a` must not move Fan B and vice-versa (addresses `0x3123` / `0x9221`).
4. **Full button sweep:** speeds 1–6, off, light toggle, CCT select, dim ±, breeze, direction.
5. **RX / state sync:** uncomment `dump: raw` under `remote_receiver`, press a factory remote,
   confirm the `claro` log line shows the right `addr`/`cmd`. Tune the mark threshold (800 µs)
   and `tolerance` if decodes are flaky. Then confirm HA fan speed / light state follows the remote.

## Known modelling limits (decided in the HA entity ticket)

- **Light on/off is a toggle with no RF feedback** → modelled as an optimistic binary light,
  corrected by the RX decoder when the physical remote is used.
- **Brightness is relative** (Dim +/- ramp) with no absolute readout → exposed as Brighter/Dimmer
  buttons, not a slider that would fake precision.
- **Colour temperature is 3 discrete presets** → a select (Warm/Natural/Cool), not a Kelvin slider.
- **Direction (F/R) is a toggle**, not absolute → tracked optimistically.
