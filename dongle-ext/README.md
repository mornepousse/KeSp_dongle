# dongle-ext — header prototype, no M.2

USB HID receiver for the **Niphargus** split keyboard, in the same shape as the
M.2 variants but without the card edge: USB arrives on a 2.54 mm header, and
the board carries its own regulator.

Prototype, kept for bench work and for hosts with no usable M.2 slot. The
designs that go in a laptop are [`../dongle-REV1/`](../dongle-REV1/) and
[`../dongle-REV2/`](../dongle-REV2/).

## Architecture

```
J1, 4-pin 2.54 mm header
  ├── VBUS 5 V ──> U1 (LD1117S33) ──> 3.3 V rail
  └── D+/D− ────> CH334R hub ──┬─> port 1: ESP32-S3 native USB (HID)
                               ├─> port 2: CH340C (UART, flash/debug)
                               └─> ports 3-4: not connected

ESP32-S3 ──SPI──┬── NRF24L01+ U2 ──2.4 GHz── Niphargus left half
                └── NRF24L01+ U3 ──2.4 GHz── Niphargus right half
```

## J1

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
| U7 | ESP32-S3-WROOM-1 | MCU, USB HID | module |
| U2, U3 | NRF24L01+ | one radio per keyboard half | breakout, 2 mm headers |
| U6 | CH334R | USB 2.0 hub, crystal-free | QSOP-16 |
| U4 | CH340C | USB-to-UART for flashing and console | SOIC-16 |
| U1 | LD1117S33 | 3.3 V from J1's VBUS | SOT-223 |
| Q1 | UMH3N | auto-reset, DTR/RTS to EN and IO0 | SOT-363 |

Q1 is a **dual** transistor: its two schematic units are one physical package.

## Differences from the M.2 variants

- **Own supply**: LD1117S33 from the header's 5 V, instead of the slot's 3.3 V.
  There is no reverse-blocking diode here — nothing can back-feed the header,
  since there is no second source.
- **R9 is still fitted**: 100 Ω between the 3.3 V rail and the CH334R's pin 12
  (5V). REV1 dropped it in V1.1 and ties pin 12 straight to 3.3 V. Either way
  the point stands — leave pin 12 floating and the part overheats and refuses
  to enumerate.
- **No M.2 control signals**: no CONFIG pins, no `W_DISABLE1#`, no `RESET#`
  into `EN`. The host cannot reset the module.

## PCB

KiCad 10. Gerbers in `Gerber/`, exported 2026-06-04.
