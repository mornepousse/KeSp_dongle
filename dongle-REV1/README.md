# dongle-REV1 — M.2 Key B receiver

USB HID receiver for the **Niphargus** split keyboard: two NRF24L01+ radios,
one per half, an ESP32-S3-WROOM-1, and an M.2 Key B card edge that plugs into a
laptop WWAN slot.

This is the design that was manufactured as **V1.0** and has been running in the
M.2 WWAN slot of a Dell Latitude 5430 for several months.

> **The files here are ahead of that board.** Since the V1.0 batch: R9 removed,
> decoupling completed (C8–C12), and the CONFIG jumpers JP2–JP5 added.
> `Gerber/` is the V1.0 export (2026-05-27); `jlcpcb/` is a later one.
> Superseded by [`../dongle-REV2/`](../dongle-REV2/), which drops the USB hub.

## Architecture

```
M.2 WWAN slot
  ├── USB 2.0, pins 7/9 ──> CH334R hub ──┬─> port 1: ESP32-S3 native USB (HID)
  │                                      ├─> port 2: CH340C (UART, flash/debug)
  │                                      └─> ports 3-4: J1, J3 headers
  ├── 3.3 V, pins 2/4/70/72/74 ─────────> everything, no onboard regulator
  └── RESET#, pin 67 ──R22 10k─────────> ESP32-S3 EN

ESP32-S3 ──SPI──┬── NRF24L01+ U2 ──2.4 GHz── Niphargus left half
                └── NRF24L01+ U3 ──2.4 GHz── Niphargus right half
```

## M.2 Key B connections

| M.2 pin | Signal | On this board |
|---|---|---|
| 1, 21, 69, 75 | CONFIG_3, _0, _1, _2 | JP5, JP2, JP3, JP4 — solder jumpers to GND, **open by default** |
| 2, 4, 70, 72, 74 | +3.3 V | main rail |
| 3, 5, 11, 27, 33, 39, 45, 51, 57, 71, 73 | GND | ground |
| 6 | `FULL_CARD_POWER_OFF#` | R23, 10 kΩ pull-up to 3.3 V |
| 7 | `USB_D+` | CH334R pin 11 (DPU+) |
| 8 | `W_DISABLE1#` | R21, 10 kΩ pull-up to 3.3 V |
| 9 | `USB_D−` | CH334R pin 10 (DMU−) |
| 67 | `RESET#` | R22, 10 kΩ into the ESP32-S3 `EN` node |
| everything else | — | not connected |

`EN` also carries a 10 kΩ pull-up (R19) and 100 nF to ground (C7), so the host
can reset the module through pin 67.

The CONFIG pins are what a strict host reads *before* powering the slot. Left
floating they read `1111`, "no add-in card present" — which is why the V1.0
board is invisible in a ThinkPad P16 Gen 1. See
[`../docs/hote-thinkpad-p16.md`](../docs/hote-thinkpad-p16.md).

## Components

| Ref | Part | Role | Package |
|---|---|---|---|
| U7 | ESP32-S3-WROOM-1 | MCU, USB HID | module |
| U2, U3 | NRF24L01+ | one radio per keyboard half | breakout, 2 mm headers |
| U6 | CH334R | USB 2.0 hub, crystal-free, 3.3 V | QSOP-16 |
| U4 | CH340C | USB-to-UART for flashing and console | SOIC-16 |
| Q1 | UMH3N | auto-reset, DTR/RTS to EN and IO0 | SOT-363 |

Q1 is a **dual** transistor: its two schematic units are one physical package.

## Power

Straight from the M.2 slot's 3.3 V rail, no onboard regulator. The CH334R runs
at 3.3 V with its internal LDO bypassed — **pin 12 (5V) and pin 13 (VDD33) both
tied to 3.3 V**. Leaving pin 12 floating makes the part overheat and blocks
enumeration; it is the classic trap with this package.

## PCB

M.2 Key B **3042 (30 × 42 mm)**, 0.8 mm thick, 4 layers. KiCad 10, exported for
JLCPCB. The NRF24L01+ modules sit on the bottom side, antennas past the board
edge. Everything else is SMD.
