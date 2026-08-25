# Research: Claro Whisper DC (C5001/101B) RF protocol — prior art

Resolves [#2](https://github.com/Adroz/claro-fan-rf/issues/2). Researched 2026-08-25.

**TL;DR:** No Claro primary source states an operating frequency, but everything points to
**433.92 MHz fixed-frequency OOK (PWM), fixed code, one discrete code per button, learn-mode
pairing**. No rtl_433 decoder or public capture exists for a Claro-branded remote, but two
fully decoded same-cluster protocols are strong candidates: the unmerged **Funpower FP0317A /
Calibo** spec (rtl_433 issue #3192, 40-bit) and the merged **`universalfanctrl`** decoder
(#286, 33-bit). One real risk: newer remotes in the same AU retail channel have moved to
2.4 GHz BLE, so confirm 433 MHz activity before investing in decode work.

---

## 1. Vendor primary sources (Claro / Universal Fans)

### CONFIRMED

- **Product page** — C5001/101B, "no-beep remote", 6 speeds, 1/2/4 hr timer, summer/winter
  reverse, dimmable CCT LED; lists Bond Bridge ($249) as the compatible smart accessory.
  <https://clarofans.com.au/product/claro-whisper-dc-low-profile-with-cct-led-light-111cm/>
- **Installation manual (VER 2.0, 16 pp)** —
  <https://clarofans.com.au/wp-content/uploads/manuals/claro-whisper_low_profile_dc-user_manual.pdf>
  - **No frequency stated anywhere.** Only RF language (p.11): *"The Hand-Held Remote-Control
    System is equipped with a learning frequency function which has code combinations to
    prevent potential interference from other remote units. The frequency on your Receiver
    and Transmitter units have been pre-set at the factory."*
  - **No remote or receiver model number printed.** Receiver is a separate canister module
    in the mounting bracket (p.7). Remote takes 2×AAA (p.2).
  - **Pairing (p.12): learn mode on power-up, no DIP switches.** Kill wall power 10 s, power
    on, then *within 5 s* hold the remote's OFF button (top-left) 5–10 s until the fan beeps.
    Multi-fan warning: any powered receiver enters learning mode and may pair to the wrong
    remote — isolate power to the other fan when pairing. (Directly relevant to our
    two-identical-fans setup.)
  - Buttons (p.13): fan ON/OFF, light ON/OFF, 6-speed dial with central Forward/Reverse,
    dim up/down, CCT (warm/natural/cool), timers **1H/2H/4H**.
  - Model family (p.14): C5000/101B, C5001/101B, C5020/101B, C5021/101B.
- **Spec sheet** — no RF or compliance info; LED module part CLSP13-FM007.
  <https://clarofans.com.au/wp-content/uploads/spec-sheets/claro-whisper-dc-low-profile-spec-sheet.pdf>
- **Company identity** — "Claro Australia" is a trading name of **Universal Fans Pty Ltd**
  (ABN 76 104 243 898), per <https://clarofans.com.au/terms-conditions/>. Universal Fans is
  also the national distributor of **Fanco** — a direct OEM-sibling signal.
- **Claro's Bond accessory page** — sells Bond Bridge **BD-1000**, listed manufacturer
  "Three Sixty" (AU distributor), states *"Works with most 300-450MHz ceiling fans"*. Does
  **not** name a Bond template or list covered Claro remotes.
  <https://clarofans.com.au/product/bond-bridge-smart-accessory/>

### UNKNOWN

- **ACMA/RCM filing** — the ACMA supplier register / EESS database is not web-searchable; no
  indexed filing found for Universal Fans Pty Ltd or C5001. Dead end without a manual DB query.
- **FCC twin** — no US-market remote with the exact button set (6-speed dial + F/R centre +
  CCT + 1H/2H/4H, no-beep) found on fccid.io / fcc.report.
- **OEM/transmitter model** — nothing printed in any Claro document; no teardown published.

## 2. Frequency & modulation

- **CONFIRMED:** Bond Bridge BD-1000 (the vendor-endorsed accessory) supports **RF 300–450 MHz**
  only (sub-GHz OOK-style learn-and-replay).
  Spec sheet: <https://bondhome-product-docs.s3.amazonaws.com/BD-1000/%5BBond+Bridge%5D+Spec+Sheet.pdf>;
  help centre: <https://olibra.zendesk.com/hc/en-us/articles/18927213862939>
- **LIKELY: 433.92 MHz, OOK, PWM symbol encoding.** Every decoded AU DC-fan lookalike
  (Fanco Flow DC, UniFan-24V, Calibo/FP0317A, Martec, Mercator, Lucci) sits at 433.92 MHz
  OOK; AU LIPD class licence makes 433.92 the default for this device class. No primary Claro
  source states it, so it must be confirmed at capture time.
- **RISK (CONFIRMED for a sibling brand): newer remotes may be 2.4 GHz BLE, not RF.**
  Three Sixty (same retail channel; the listed "manufacturer" of Claro's Bond Bridge SKU)
  ships newer remotes on 2.4 GHz that Bond cannot learn — Bond forum:
  <https://forum.bondhome.io/t/problem-linking-threesixty-ceiling-fan-to-bond-bridge-pro/5436>.
  Some newer Fanco remotes reported BLE too:
  <https://community.home-assistant.io/t/help-on-rtl-433-reading-the-same-code-for-different-buttons-fanco-fan-remote-control/594761>.
  First capture step must therefore be "is there any 433.92 MHz energy at all when I press a
  button" — if silent, sweep 300–450 MHz, then check BLE advertisements.

## 3. rtl_433 coverage

- **CONFIRMED: no merged decoder, conf, or test capture mentions Claro, Fanco, Hunter
  Pacific, or Three Sixty** (empty results in rtl_433 issue/PR search and `src/devices` /
  `conf/` listings). <https://github.com/merbanan/rtl_433>

### Best-match candidate: Funpower FP0317A (unmerged, rtl_433 issue #3192) — LIKELY same OEM family

- Issue (Calibo Smart CloudFan "and many other brands"):
  <https://github.com/merbanan/rtl_433/issues/3192>; full protocol spec attachment:
  <https://github.com/user-attachments/files/30159278/index.md>
- 433.92 MHz, OOK/ASK PWM. Leader 300 µs high + 5100 µs low; `0` = 300/900 µs,
  `1` = 900/300 µs (measured ~210/~810–840 µs); ≥5 bursts per press (~250 ms).
- **40-bit payload: 16-bit remote ID | type byte (always `0x17`) | button code | action-flags
  byte** (bit7 hold, bit6 release, low bits mirror key code).
- Button map: speeds 1–6 = `1c,1b,17,18,19,1d`; fan on/off `1a`; direction `1e`; breeze `0e`;
  light on/up/down/off `12/11/13/10`; sleep 1H/4H/8H `14/15/16`; **beep on/off toggle** (hold
  sleep-1H within 20 s of power-on) — consistent with Claro's "no-beep" marketing.
- The (older, non-low-profile) Claro Whisper DC manual
  (<https://clarofans.com.au/wp-content/uploads/2020/08/claro-whisper-manual.pdf>, CFCLWH3DC*)
  shows the same button set incl. **1H/4H/8H** timers and the same pairing gesture — a strong
  behavioural match to FP0317A. **Caution:** our low-profile C5001 remote has **1H/2H/4H**
  timers, so it is a sibling, not necessarily bit-identical.
- Same-OEM corroboration: open issue #3345, Sofucor remote **FT0317R** from "Shenzhen
  Funpower General Technology", near-identical timings (302/906 µs PWM):
  <https://github.com/merbanan/rtl_433/issues/3345>
- Flex specs published by maintainers:
  - `-X 'n=calibo,m=OOK_PWM,s=300,l=900,g=1500,r=6000'` (#3192)
  - `-X 'n=FanRemote,m=OOK_PWM,s=300,l=900,g=1000,r=5000'` (#3345)

### Second candidate: `universalfanctrl` — merged decoder #286 — LIKELY family, feature near-match

- Source: <https://github.com/merbanan/rtl_433/blob/master/src/devices/universalfanctrl.c>
  (PR <https://github.com/merbanan/rtl_433/pull/3142>, "Universal (Reversable) 24V Fan
  Controller" / Eubea DC fans).
- 433.92 MHz OOK PWM: s=256, l=756, **sync 3616 µs sent once**, 7 repeated 33-bit frames,
  ~8.2 ms inter-frame gaps.
- **33-bit frame: 20-bit address | 5-bit button | 3-bit press counter (cosmetic, replays with
  any value accepted) | 4-bit XOR checksum (folds to 0xA) | fixed 1.**
- Buttons (issue <https://github.com/merbanan/rtl_433/issues/3648>): All Off=0x19,
  Light=0x17, Fwd=0x1B, Rev=0x0E, Breeze=0x0A, Fan Off=0x09, Speeds 1–6 =
  0x0F/0x0D/0x03/0x15/0x10/0x13, Timers 1H/2H/3H = 0x1D/0x16/0x06.
- Runs automatically in stock `rtl_433 -f 433.92M` — no flex spec needed to test it.

### Other decoded AU fan protocols (not the Claro protocol, useful reference)

- **Fanco Flow DC** (same distributor as Claro!) fully decoded:
  <https://github.com/peterdev22/fanco-flowdc-rf> — 433.92 MHz OOK PWM, ~1200 µs bit period
  (`0` = 387/801 µs, `1` = 935/230 µs, EOF 347 µs + 4326 µs gap, ×5 repeats); **32-bit frame =
  command byte repeated twice + 16-bit fan ID**; fixed code. Commands: fan power 0x11,
  speeds 1–6 = 0x02–0x07, breeze 0x0F, timers 1H/4H/8H = 0x00/0x01/0x0E, direction 0x09,
  light 0x0A, brighten 0x0C, dim 0x0D.
- **Martec MPLCD** (AU, 3-speed AC): `martec_mplcd.c`, 433.92 MHz, s=292/l=648, 22-bit.
- **Mercator FRM87GL** (AU, AC): `conf/Mercator.conf`, 433.92 MHz OOK_PCM.
- **Lucci Air / Beacon Lighting** (AU, AC 3-speed): capture only, in rtl_433_tests
  (<https://github.com/merbanan/rtl_433_tests/tree/master/tests/lucci-air-ceiling-fan-remote>).
- US-market fan decoders sit at 303–315 MHz (regency, FAN-11T, Hampton Bay, Honeywell) —
  irrelevant except as fallback sweep frequencies.

## 4. Bond database detail

- **UNKNOWN: no named Bond template for Claro.** Zero "Claro" hits on forum.bondhome.io;
  Bond's supported-devices lookup (<https://bondhome.io/supported-devices/>) is FCC-ID keyed,
  so AU-only remotes never appear. Claro fans would use Bond's generic **learn-from-remote
  ("Non-Smart Ceiling Fan") signal-recording flow**, not a curated template. The useful
  takeaway is the confirmed one: Bond only records fixed-frequency 300–450 MHz signals, and
  the vendor endorses it — so the protocol is a replayable fixed code in that band.
- **CONFIRMED by proxy (siblings): fixed code, learnable, replayable.** Bond learns Hunter
  Pacific ("open source RF" per manufacturer, Whirlpool:
  <https://forums.whirlpool.net.au/archive/9qr1z2vl>) and older Three Sixty remotes;
  Broadlink RM4 Pro learn-and-replay works on Fanco Sanctuary DC
  (<https://community.home-assistant.io/t/adventures-with-fanco-ceiling-fans-localtuya/986680>).
  Impossible with rolling code. Ventair is the only reported Bond-incompatible AU brand
  ("proprietary RF encoding" — still fixed, just non-standard).

## 5. Community decode prior art / gotchas

- **CONFIRMED: nobody has published a capture of a Claro-branded remote** — HA forum,
  GitHub, Reddit, Whirlpool, EEVblog all empty for "Claro" + fan. We would be first.
- **Replay gotcha #1 (CONFIRMED):** UniFan-family receivers reject retransmissions that
  insert the sync pulse before *every* frame repeat; the real remote sends sync once, then
  back-to-back frames. rtl_433 decoding your own transmission as checksum-valid does **not**
  mean the fan will accept it.
  <https://community.home-assistant.io/t/replaying-a-universal-24v-fan-controller-rtl-433-decoder-286-remote-fan-doesnt-respond-despite-clean-checksum-valid-transmissions/1020267>
- **Replay gotcha #2 (CONFIRMED, same thread):** cheap STX882/FS1000A TX modules measured
  37–55 kHz off 433.92 MHz and the receiver ignored them; a CC1101 (crystal-disciplined,
  frequency-settable) fixed it. Validates the ESP32+CC1101 plan.
- **Decoder-aliasing gotcha (CONFIRMED):** rtl_433's generic decoders can collapse distinct
  buttons into one code (Fanco Galaxy thread above) — verify distinct payloads in the raw
  pulse view (`-A` / triq.org/pdv) before concluding two buttons share a code.
- **Multi-burst capture gotcha:** long multi-burst presses can overflow naive ESP32 RMT
  capture; capture with the RTL-SDR, transmit in split sequences if needed.
  <https://community.home-assistant.io/t/esp32-cc1101-and-my-last-stubborn-fan/1012598>
- **Per-fan addressing (feeds map-issue question):** every candidate protocol carries a
  16–20-bit transmitter ID and receivers pair to it via learn mode — so the two identical
  fans are individually addressable as long as they were paired to different remotes
  (LIKELY; verify during capture that the two remotes emit different IDs).

## 6. Recommendations for the capture session

1. **Confirm RF exists at all:** `rtl_433 -f 433.92M -A` (or gqrx/SDR++ waterfall at
   433.92 MHz) while pressing remote buttons. If silent, sweep 300–450 MHz (try 315M, 303.9M),
   and if still silent, scan BLE advertisements (nRF Connect on a phone) — the
   newer-generation-is-BLE risk is real in this retail channel.
2. **Let stock decoders try first:** `rtl_433 -f 433.92M -S unknown` — if the remote is
   UniFan-family, merged decoder #286 (`universalfanctrl`) fires with no extra flags.
3. **Then try flex specs, in this order:**
   - FP0317A/Calibo: `rtl_433 -f 433.92M -X 'n=claro,m=OOK_PWM,s=300,l=900,g=1500,r=6000'`
     — compare against the 40-bit `ID:16 | 0x17 | CMD:8 | FLAGS:8` layout and #3192 button map.
   - Funpower FT0317R variant: `-X 'n=claro,m=OOK_PWM,s=300,l=900,g=1000,r=5000'`
   - Fanco Flow timings (387/935 µs, 32-bit `CMD CMD ID ID`) if the above misfit.
4. **Analyze raw captures** with `rtl_433 -A g*.cu8` and <https://triq.org/pdv/> before
   inventing a new decoder; save all `.cu8` files.
5. **Capture every button from BOTH remotes** (all 6 speeds, F/R, light, dim up/down held +
   tapped, CCT, all timers, off) — needed for per-fan ID comparison and to see whether
   dim/CCT are discrete-value or cycle commands (open map-issue question).
6. **For transmit later:** use the CC1101 (not a bare 433 TX module), replicate the burst
   structure exactly (sync-once, N back-to-back repeats, correct inter-burst gaps).
7. **Upstream:** if the capture matches FP0317A, contribute the capture to rtl_433 issue
   #3192 — it directly advances the pending decoder, and a merged decoder gives us the
   permanent rtl_433→MQTT receive path mooted for state sync.
