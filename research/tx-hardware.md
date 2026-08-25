# Research: choosing a 433.92 MHz OOK transmitter for the Claro fans

Resolves [#8](https://github.com/Adroz/claro-fan-rf/issues/8).

## The question

Which specific 433.92 MHz transmitter should the owner (Australia, 2026) buy to drive
the Claro Whisper DC ceiling fans from Home Assistant OS, ideally plugged into the HAOS
host via USB, with enough range to punch through one plaster/stud interior wall?

The device **must transmit arbitrary raw OOK** — our decoded protocol is not a known
vendor protocol, so anything locked to a fixed protocol database is useless. From
[`PROTOCOL.md`](../PROTOCOL.md): 433.92 MHz OOK/PWM, short pulse ~412 µs, long pulse
~1192 µs, frame = `[16-bit addr][8-bit cmd][8-bit chk=addr_hi^addr_lo^cmd][1 stop bit]`
(33 bits, ~66 pulse edges), ~4 repeats, ~11.9 ms inter-frame gap, fixed/replayable code.

Owner already has a spare **ESP32**; the nRF24L01 they found is unusable (2.4 GHz).

Confidence tags used below: **CONFIRMED** (verified against a primary/first-party
source), **LIKELY** (strong secondary evidence or sound inference), **UNKNOWN** (could
not verify).

---

## TL;DR recommendation

**Build the ESP32 + CC1101 + ESPHome transmitter and place it in the same room as the
fans (WiFi back to HA).** This sidesteps both hard constraints: putting the radio near
the fans deletes the through-wall problem, and ESPHome's `remote_transmitter.transmit_raw`
sends genuinely arbitrary OOK timings — a perfect match for our custom frame. The owner
already has the ESP32, so the only purchase is a ~AUD $5–12 CC1101 module and a straight
antenna (or a free 16.4 cm wire).

The literal "USB into the HAOS host" request is achievable **only** with an
**RFXtrx433e/XL flashed to Pro firmware** (~AUD $80–180), and even then you hand-build raw
hex packets and must flash the Pro firmware on a separate PC first. It is the right answer
only if keeping everything physically on the HAOS box matters more than cost/simplicity —
and it still has to cross the wall, which the ESP option avoids.

---

## Part 1 — USB-to-HAOS radios: can each emit arbitrary raw OOK?

The recurring HAOS constraint: **HAOS is a locked appliance OS.** No host shell, no
serial access, no `pip install`/CLI against `/dev/ttyUSB0`. You are limited to (a) core
integrations, (b) HACS custom integrations, (c) add-ons. Anything that needs raw bytes
poked at a serial port from a terminal is effectively blocked. Firmware flashing of any of
these devices also cannot be done from HAOS — it needs a separate PC.

| Device | Arbitrary raw OOK from HAOS? | Verdict |
|---|---|---|
| RFXtrx433e/XL (Pro firmware) | Yes — `rfxtrx.send` passes raw bytes; Pro-fw raw-TX packet fits our frame | **CONFIRMED** (one-time PC flash) |
| Homeduino (HACS) | Yes — `homeduino.raw_rf_send` takes a custom pulse train | **CONFIRMED** (DIY build + flash) |
| nanoCUL / CUL (culfw) | No practical path | **UNKNOWN → not viable** |
| RFLink | No — named protocols only | **CONFIRMED not possible** |
| rtl_433 / RTL-SDR | No — receive-only hardware | **CONFIRMED not possible** |

### RFXtrx433e / RFXtrx433XL (RFXCOM) — `rfxtrx` integration — CONFIRMED (with firmware caveat)

This is the only "buy it, plug into HAOS USB" path that works end-to-end.

- The RFXCOM **Pro firmware can transmit RAW data**: space-delimited pulse lengths in µs,
  an even number of pulses, **max 142 pulses**, max single pulse **65535 µs**, with a
  repeat count. Our frame is ~66 pulse edges and 412/1192 µs pulses — comfortably inside
  those limits. Source: openHAB RFXCOM binding reference (documents the same firmware
  raw-transmit packet) https://www.openhab.org/addons/bindings/rfxcom/ and the RFXtrx User
  Guide "Receive And Transmit Raw Data"
  https://www.manualslib.com/manual/1970493/Rfxcom-Rfxtrx-Series.html?page=51 — **CONFIRMED**.
- HA passes arbitrary raw bytes straight through: the `rfxtrx.send` action does no protocol
  validation — it hex-decodes the string and writes it to the transport. Source:
  `homeassistant/components/rfxtrx/services.py`
  https://raw.githubusercontent.com/home-assistant/core/dev/homeassistant/components/rfxtrx/services.py
  and the action doc https://www.home-assistant.io/actions/rfxtrx.send/ — **CONFIRMED**.
  Community example of sending raw hex via this service:
  https://community.home-assistant.io/t/rfxtrx-to-send-a-raw-package/193757
- **Caveat / HAOS friction:** raw TX exists only in the **Pro** firmware
  (RFXtrx433E-Pro/Pro2, ProXL on the 433XL); stock firmware does not do raw transmit.
  Flashing is done with RFXCOM's desktop tool on a PC — **not possible from HAOS**. And
  there is no HA UI helper: you hand-compute the `0x7F` raw pulse-list hex yourself.
  Sources: https://rfxcom.com/en/updates/updated-firmware-rfxtrx433e-pro2-rfxtrx433xl ,
  https://rfxcom.com/en/updates/rfxtrx433xl-information ;
  hand-built raw packets on sibling ecosystems:
  https://forum.domoticz.com/viewtopic.php?t=27260 — **CONFIRMED** raw TX capability,
  **LIKELY** on exactly which current SKUs accept Pro firmware (verify with RFXCOM before
  buying).
- **AU price:** RFXCOM dual-band USB transceiver ~AUD $80 at Black Cat Control Systems
  https://www.blackcatcontrolsystems.com.au/RFX433-USB ; also Smart Living AU
  https://www.smartliving.com.au/rfxcom (a listing there shows AUD $198 for a bridge SKU).
  Genuine RFXtrx433e/Pro2 imported from EU is ~AUD $150–180 landed. **LIKELY.**

### Homeduino — HACS `homeduino` integration — CONFIRMED (DIY hardware)

- The HACS integration exposes **`homeduino.raw_rf_send` — "raw RF command for unsupported
  protocols"**, accepting a custom pulse train (space-delimited µs lengths). HACS installs
  on HAOS, so the raw-send path is reachable without a shell. Source:
  https://github.com/rrooggiieerr/homeassistant-homeduino — **CONFIRMED**.
- **Caveat:** not a dongle — it's an **Arduino Nano flashed with the homeduino sketch**
  plus a 433 MHz TX module; one-time flash on a PC. Sketch:
  https://github.com/pimatic/homeduino — **CONFIRMED**. Cheapest in parts, but you build it.

### nanoCUL / CUL (culfw) — UNKNOWN → not viable on HAOS

- **No official HA integration**; consensus is to bridge via FHEM/Homegear, which speak
  specific device protocols, not arbitrary raw. Source:
  https://community.home-assistant.io/t/usb-nanocul-868mhz-433-mhz/264092 — **CONFIRMED**
  (no direct integration).
- culfw's raw send (`G`, compiled only with `HAS_RAWSEND`) is Manchester/preamble-oriented,
  not free-form short/long OOK. A hands-on thread concluded culfw lacks "a generic function
  to simply alternate output with raw timings provided via tty," and reproducing arbitrary
  pulse trains needed **custom firmware modification**. Command ref:
  http://culfw.de/commandref.html (broken TLS; mirrored in FHEM `00_CUL.pm`
  https://github.com/mhop/fhem-mirror/blob/master/fhem/FHEM/00_CUL.pm); analysis:
  https://groups.google.com/g/cul-fans/c/B3nXuV_4xKY — **LIKELY** inadequate. Combined with
  HAOS blocking raw serial: **not a realistic path.**
- **AU price:** no mainstream AU stock; EU/eBay ~AUD $55–80.

### RFLink — `rflink` integration — CONFIRMED cannot send arbitrary raw OOK

- Transmit is strictly protocol-name based: `10;ProtocolName;address;switch;action;`. No
  raw/arbitrary-pulse transmit; `RFDEBUG` only affects *receive* logging. Source:
  https://rflink.nl/protref.php ; HA integration models only predefined device types
  https://www.home-assistant.io/integrations/rflink/ — **CONFIRMED not suitable.**

### rtl_433 / RTL-SDR — CONFIRMED receive-only

- RTL-SDR dongles are receive-only; rtl_433 has no TX. Source:
  https://github.com/merbanan/rtl_433/discussions/2941 — **CONFIRMED cannot transmit.**

---

## Part 2 — ESP32/ESP8266 + CC1101 via ESPHome (over WiFi)

**Bottom line: CONFIRMED viable today.** Use the native `cc1101` component (ESPHome
2025.12.0) as an RF frontend behind `remote_transmitter.transmit_raw`. There is one open
TX gotcha with a known workaround, and the exact ceiling-fan use case is documented working.

### Native `cc1101` component — CONFIRMED

- Async mode "integrates with the Remote Transmitter and Remote Receiver components for
  encoding and decoding RF protocols" — i.e. CC1101 as an OOK frontend, timings produced by
  `remote_transmitter`. Default modulation is **ASK/OOK**. Source:
  https://esphome.io/components/cc1101/ — **CONFIRMED**.
- Added in **ESPHome 2025.12.0** (Dec 2025). Sources:
  https://esphome.io/changelog/2025.12.0/ ,
  https://community.home-assistant.io/t/using-the-new-cc1101-component/980890 — **CONFIRMED**.
- Config: `cs_pin`, `frequency` (300–928 MHz, set `433.92MHz`), `output_power` (−30 to
  **11 dBm**, default 10), `symbol_rate`, `gdo0_pin` (TX data line — CC1101 only accepts TX
  input on GDO0/module pin 3), optional `gdo2_pin`, plus the standard `spi:` bus. Source:
  https://esphome.io/components/cc1101/ — **CONFIRMED**.
- `remote_transmitter.transmit_raw` sends a list of µs timings (positive = carrier on,
  negative = off). For RF use `carrier_frequency: 0Hz`, `carrier_duty_percent: 100%`, and
  `repeat: { times: N, wait_time: … }` for repeats/gap — maps cleanly onto 412/1192 µs,
  ~4 repeats, 11.9 ms gap. Source: https://esphome.io/components/remote_transmitter/ —
  **CONFIRMED**. (The remote_* pages don't themselves mention CC1101; the integration is
  documented on the cc1101 page.)

### The `gdo0_pin` TX quirk (issue #16876) — CONFIRMED (open bug, workaround exists)

- On ESP32, `remote_transmitter` drives GDO0 via the **RMT peripheral**, but the cc1101
  component's `pin_mode()` calls re-route that pad to plain GPIO, severing the
  RMT→GPIO-matrix→GDO0 path. The chip enters TX mode but the carrier is never keyed → **no
  RF output**. Source: https://github.com/esphome/esphome/issues/16876 — **CONFIRMED**,
  status OPEN, no merged fix.
- **Workaround (CONFIRMED):** remove the `gdo0_pin:` block from the `cc1101:` config; the
  null-checks then skip the offending `pin_mode()` calls and RMT keeps its routing.
  `remote_transmitter` still targets the same physical GPIO wired to GDO0.
- ESP8266 uses a different TX path, so the quirk may not apply there — **UNKNOWN**, verify
  separately if using ESP8266. (Owner has an ESP32, so this workaround applies.)

### RadioLib alternative (`juanboro/esphome-radiolib-cc1101`) — CONFIRMED but secondary

- External component that also does arbitrary raw OOK TX; predates the native component. Its
  own README now says to use the built-in CC1101 component unless you want RadioLib's extras
  (bandwidth/data-rate/AGC controls, UDP dump, rtl_433 decode). Source:
  https://github.com/juanboro/esphome-radiolib-cc1101 — **CONFIRMED**. Keep as fallback only.

### Output power — CONFIRMED

- TI CC1101 datasheet: "**Programmable output power up to +12 dBm**"; the flat +12 dBm is
  the cross-band headline, and at 433 MHz practical max is **~+10 to +11 dBm** (top PATABLE
  entry). +20/+27 dBm only apply with the separate CC1190 range extender, not a bare CC1101.
  Source: https://www.ti.com/lit/ds/symlink/cc1101.pdf (p.2) — **CONFIRMED** headline,
  **LIKELY +10 dBm** at 433. ESPHome exposes `output_power:` up to 11, so set it in YAML.

### Architecture implication — CONFIRMED

- Placing ESP32+CC1101 near the fans removes the through-wall problem entirely: ESPHome
  talks to HA over WiFi/LAN, and the only RF hop is the short CC1101→fan link. Standard
  ESPHome deployment; no special support needed.
- **The exact use case is documented working:** victorchang.codes "Controlling Ceiling Fans
  with Home Assistant — Round 2" drives ESP32 + CC1101 + ESPHome, transmitting **raw OOK**
  via `remote_transmitter` (a lambda pushes an `int32_t` timing array into `RawTimings`,
  `carrier_duty_percent: 100%`, wrapping bursts in `cc1101.begin_tx`/`end_tx`). Source:
  https://victorchang.codes/controlling-ceiling-fans-with-home-assistant-part-2 —
  **CONFIRMED** arbitrary raw OOK TX works end-to-end. Corroboration:
  https://community.home-assistant.io/t/cc1101-rf-transmitting/1007183

---

## Part 3 — Range physics through one plaster wall + antenna choice

**Antenna quality/matching/ground plane dominates, not a few dBm of TX power.**

- A single plaster/stud interior wall is a small obstacle: ~4.4 dB for a two-sheet drywall
  + wood-stud office wall at 900 MHz (each drywall sheet ~0.8 dB), and **lower at 433 MHz**
  (longer wavelength penetrates better). Concrete is the expensive case (102 mm ≈ 12 dB,
  305 mm ≈ 35 dB). Source: Digi Indoor Path Loss App Note XST-AN005a, Table 2
  https://ftp1.digi.com/support/images/XST-AN005a-IndoorPathLoss.pdf — **CONFIRMED**.
- Link margin = TX power − RX sensitivity + antenna gain − path loss; **"every 6 dB of link
  margin doubles the range."** A cheap coil/"spring" antenna is easily 6–10 dB worse than a
  proper quarter-wave (electrically short, loading-coil loss, no ground plane) — a bigger
  swing than jumping +7→+10 dBm TX (3 dB). Same Digi app note — **CONFIRMED / LIKELY**.
- **Quarter-wave length at 433.92 MHz:** λ = c/f = 0.6909 m → λ/4 = **0.1727 m ≈ 17.3 cm**
  (free-space/electrical). With wire velocity factor ~0.95, cut physical ≈ **16.4 cm**.
  Calculators return ~16.5 cm for this reason. Sources:
  https://m0ukd.com/calculators/quarter-wave-ground-plane-antenna-calculator/ ,
  https://www.66pacific.com/calculators/quarter-wave-vertical-antenna-calculator.aspx —
  **CONFIRMED**.
- Antenna ranking for one-wall indoor: straight quarter-wave (17.3 cm wire) ≈ straight SMA
  whip **>>** tiny coil "spring". Wikipedia (Whip antenna): "all these electrically short
  whips have lower gain than a full-length quarter-wave whip," and quarter-wave monopoles
  need a conducting ground plane. Source: https://en.wikipedia.org/wiki/Whip_antenna —
  **CONFIRMED**. A correctly-cut 16.4 cm straight wire soldered to ANT is genuinely
  competitive with a paid SMA whip; the connector mostly buys durability/repeatability, not
  RF. Hobbyist consensus: https://forum.arduino.cc/t/433-mhz-antenna-options/848403 —
  **CONFIRMED / LIKELY**.

**Takeaway:** don't chase a high-power PA module for one drywall wall — spend the effort on
a straight resonant quarter-wave with a ground plane, keep it straight and clear of metal.
(And with the recommended ESP-near-the-fans placement, the wall is a non-issue anyway.)

---

## Part 4 — Concrete AU-available products (2026)

Legality: **433.05–434.79 MHz is licence-free in Australia** under the ACMA LIPD class
licence; max EIRP **25 mW (~+14 dBm)**, shared/no-protection basis. Sources:
https://www.acma.gov.au/licences/low-interference-potential-devices-lipd-class-licence ,
https://en.wikipedia.org/wiki/LPD433 — **CONFIRMED** (exact 25 mW line **LIKELY**). The
+10 dBm CC1101 is comfortably legal; any +20/+27 dBm PA module must be turned down to ≤25 mW.

**CC1101 modules**

| Board | Connector | TX power | Rough AUD | AU availability | Source |
|---|---|---|---|---|---|
| Ebyte **E07-M1101D-SMA** | SMA | +10 dBm | ~$5–8 + ship | AliExpress/eBay → AU | https://www.aliexpress.com/item/32975228488.html |
| Ebyte **E07-M1101D-TH** | DIP + spring | +10 dBm | ~$4–7 | AliExpress/eBay | https://www.cdebyte.com/products/E07-M1101D-TH/2 |
| Generic CC1101 433 MHz module | bare/spring/SMA | +10 dBm | **$11.90** (AU stock) | Little Bird Electronics (AU) | https://littlebirdelectronics.com.au/products/433mhz-rf-transceiver-cc1101-module |
| Ebyte **E07-433M20S** (PA/LNA) | IPEX/SMA | +20 dBm | ~$12–18 | AliExpress → AU | https://ebyteiot.com/collections/rf-module/products/ebyte-e07-433m20s-cc1101-433mhz-20dbm-wireless-transceiver-module-smart-home-spi-interface-power-amplifier-rf-receiver-module |

Note: the +20 dBm E07-433M20S exceeds the 25 mW LIPD ceiling at full power — legal to buy,
but turn output down to comply. For one wall (or ESP-near-fans placement) the plain +10 dBm
E07-M1101D is plenty. "E07-900MM10S" is a 900 MHz part, not 433 — **CONFIRMED** correction.

**Antenna**

- SparkFun 2 dBi straight SMA whip, 433 MHz — **AUD $32.95** at Core Electronics
  https://core-electronics.com.au/sparkfun-antenna-sma-2dbi-433mhz.html — **CONFIRMED**.
- Adafruit spring antenna (~38 mm coil) — AUD $2.05
  https://core-electronics.com.au/simple-spring-antenna-433mhz.html — **CONFIRMED** (but the
  weakest performer; a free 16.4 cm wire beats it).

**USB stick (for the USB-to-HAOS path)**

- RFXCOM dual-band USB transceiver ~AUD $80 (Black Cat)
  https://www.blackcatcontrolsystems.com.au/RFX433-USB ; genuine RFXtrx433e/Pro2 imported
  ~AUD $150–180. **CONFIRMED available in AU / LIKELY on price.** Confirm the SKU runs Pro
  firmware before buying.

**ESP32 (owner already has one)**

- Core Electronics ESP32-S3-DevKitC-1 — AUD $57.10; generic ESP32-WROOM DevKits ~AUD $15–30
  on eBay AU/AliExpress. https://core-electronics.com.au/esp32-s3-devkitc-1-development-board.html
  — **CONFIRMED / LIKELY**.

---

## Ranked recommendation

1. **ESP32 + CC1101 (Ebyte E07-M1101D) + ESPHome, placed near the fans — RECOMMENDED.**
   Owner already has the ESP32; buy a ~AUD $5–12 CC1101 module and either cut a free 16.4 cm
   quarter-wave wire or add the AUD $33 SparkFun SMA whip. Native `cc1101` component
   (ESPHome ≥ 2025.12.0), `frequency: 433.92MHz`, `output_power: 11`, send the frame via
   `remote_transmitter.transmit_raw` (`carrier_frequency: 0Hz`,
   `carrier_duty_percent: 100%`, `repeat: {times: 4, wait_time: 11.9ms}`). Apply the
   issue #16876 workaround: **do not declare `gdo0_pin:` inside the `cc1101:` block.**
   Total new spend ~AUD $10–45. Removes the through-wall constraint entirely and does
   arbitrary raw OOK natively.

2. **RFXtrx433e/XL flashed to Pro firmware — the true USB-to-HAOS option.** Plugs into the
   HAOS host; `rfxtrx.send` pushes your raw 0x7F pulse-list hex straight through. ~AUD
   $80–180. Downsides: one-time Pro-firmware flash on a separate PC, hand-built raw hex (no
   UI helper), and it still has to cross the wall from wherever the HAOS box lives. Choose
   this only if physically co-locating the radio with HA outweighs cost/simplicity.

3. **Homeduino (Arduino Nano + 433 TX module, HACS).** Cheapest in raw parts and does
   arbitrary raw OOK from HAOS via `homeduino.raw_rf_send`, but you assemble and flash the
   Arduino yourself. A DIY sidegrade to the ESP path with no real advantage over it here.

4. **nanoCUL / RFLink / RTL-SDR — rejected.** nanoCUL has no viable HAOS raw-OOK path;
   RFLink transmits only named protocols; RTL-SDR is receive-only.

## The core tradeoff: USB-to-HAOS vs ESP-over-WiFi

| | RFXtrx433e Pro (USB→HAOS) | ESP32 + CC1101 (WiFi) |
|---|---|---|
| Physically on HAOS host | Yes (the literal ask) | No — separate WiFi node |
| Through-wall range | Must cross the wall | **Eliminated** (radio sits by the fans) |
| Arbitrary raw OOK | Yes (Pro fw + `rfxtrx.send`) | Yes (`transmit_raw`) |
| Cost (new spend) | ~AUD $80–180 | ~AUD $10–45 (ESP already owned) |
| Setup friction | PC firmware flash + hand-built hex | Wire SPI, flash ESPHome, apply #16876 workaround |
| HAOS-appliance risk | None once flashed | None (standard ESPHome) |

If the owner genuinely needs everything on the HAOS box, the RFXtrx433e-Pro is the only
confirmed answer. For everyone else — especially given the spare ESP32 and the wall —
**the ESP32 + CC1101 node next to the fans is cheaper, sidesteps the range problem, and
transmits arbitrary raw OOK just as well.**
