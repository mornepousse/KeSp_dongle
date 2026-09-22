# dongle-REV2 — M.2 Key B receiver, no hub

USB HID receiver for the **Niphargus** split keyboard: two NRF24L01+ radios,
one per half, an ESP32-S3-WROOM-1, and an M.2 Key B card edge that plugs into a
laptop WWAN slot.

Current design, **not manufactured yet**. It starts from
[`../dongle-REV1/`](../dongle-REV1/) and answers the ThinkPad P16 Gen 1, whose
WWAN slot never powers the V1.0 board. The full story, with the measurements
still to make, is in [`../docs/hote-thinkpad-p16.md`](../docs/hote-thinkpad-p16.md).

## What changed from REV1

| Change | Why |
|---|---|
| `JP2`–`JP5`: CONFIG_0..3 to GND, **open** by default | REV1 leaves the CONFIG pins floating, so a host that reads them before powering the slot sees "no add-in card present". A solder blob now declares a code, and the codes can be swept per host without a new PCB |
| CH334R hub (U6) removed, with C6 and J3 | The host sees a single device whose VID/PID the firmware owns; also retires the CH334R 3.3 V trap |
| ESP32-S3 straight onto M.2 pins 7/9 | The USB 2.0 link with nothing in between |
| CH340C moved to J1 | Serial console and auto-reset kept, as a bench programming port |
| U1 (LD1117S33) behind D1 (SS14), with C13/C14 | Runs the board from J1's VBUS. The series Schottky blocks the slot's 3.3 V from feeding back to the connector through the regulator's parasitic path |
| `JP6`, **open** by default | Isolates the LDO output from the 3.3 V rail, so the card cannot power the laptop's WWAN rail with the bench cable plugged in |

## Architecture

```
M.2 WWAN slot
  ├── USB 2.0, pins 7/9 ───────> ESP32-S3 native USB (HID), no hub
  ├── 3.3 V, pins 2/4/70/72/74 ─> main rail
  └── RESET#, pin 67 ──R22 10k─> ESP32-S3 EN

J1, bench port ── VBUS ──> D1 (SS14) ──> U1 (LD1117S33) ──JP6──> 3.3 V rail
                └─ D+/D− ──> CH340C ──UART──> ESP32-S3

ESP32-S3 ──SPI──┬── NRF24L01+ U2 ──2.4 GHz── Niphargus left half
                └── NRF24L01+ U3 ──2.4 GHz── Niphargus right half
```

Only one of the two supplies should be live at a time. `JP6` open is the
default; close it only when running off the bench cable with the card out of
the slot.

## M.2 Key B connections

| M.2 pin | Signal | On this board |
|---|---|---|
| 1, 21, 69, 75 | CONFIG_3, _0, _1, _2 | JP5, JP2, JP3, JP4 — solder jumpers to GND, **open by default** |
| 2, 4, 70, 72, 74 | +3.3 V | main rail |
| 3, 5, 11, 27, 33, 39, 45, 51, 57, 71, 73 | GND | ground |
| 6 | `FULL_CARD_POWER_OFF#` | R23, 10 kΩ pull-up to 3.3 V |
| 7 | `USB_D+` | ESP32-S3 GPIO20 (U7:14) |
| 8 | `W_DISABLE1#` | R21, 10 kΩ pull-up to 3.3 V |
| 9 | `USB_D−` | ESP32-S3 GPIO19 (U7:13) |
| 67 | `RESET#` | R22, 10 kΩ into the ESP32-S3 `EN` node |
| everything else | — | not connected |

`EN` also carries a 10 kΩ pull-up (R19) and 100 nF to ground (C7), so the host
can reset the module through pin 67.

## J1 — bench port

| Pin | Signal |
|---|---|
| 1 | GND |
| 2 | USB D− |
| 3 | USB D+ |
| 4 | VBUS, 5 V |

**This is the reverse of a USB-A cable's pin order.** Build the pigtail to
match, or 5 V lands on ground.

## Components

| Ref | Part | Role | Package |
|---|---|---|---|
| U7 | ESP32-S3-WROOM-1 | MCU, USB HID, native USB on the M.2 link | module |
| U2, U3 | NRF24L01+ | one radio per keyboard half | breakout, 2 mm headers |
| U4 | CH340C | USB-to-UART on the bench port | SOIC-16 |
| U1 | LD1117S33 | 3.3 V from J1's VBUS | SOT-223 |
| D1 | SS14 | 40 V / 1 A Schottky, blocks reverse current into J1 | SMA |
| Q1 | UMH3N | auto-reset, DTR/RTS to EN and IO0 | SOT-363 |
| JP2–JP5 | — | M.2 CONFIG code | solder jumpers |
| JP6 | — | LDO output isolation | solder jumper |

Q1 is a **dual** transistor: its two schematic units are one physical package.

## Flashing

In the slot, through the ESP32-S3's own USB Serial/JTAG on the M.2 link — its
CDC-ACM "supports host controllable chip reset and entry into download mode"
(ESP32-S3 TRM v1.8, p. 1244). Note that the internal USB PHY is **shared**
between USB-OTG and USB Serial/JTAG, one at a time (p. 1244-1245): while the
firmware holds the PHY in OTG for HID, that console is gone.

On the bench, through J1 and the CH340C, which keeps a real UART console
whatever the PHY is doing.

## PCB

M.2 Key B **3042 (30 × 42 mm)**, 0.8 mm thick, 4 layers. KiCad 10. Production
files in `jlcpcb/`; there is deliberately no `Gerber/` folder, so no stale V1.0
export can be sent to a fab by mistake.

Checked with `kicad-cli` on 2026-09-22: 0 schematic/PCB parity issues, 0
unconnected items. Known leftovers are listed in the repository's
[`../README.md`](../README.md) — a dangling via and track, two real clearance
violations, and no clearance exception on J2 yet.
