# KaSe v2 -- USB Dongle

USB dongle that receives keystrokes from both keyboard halves via NRF24L01+ and presents as USB HID to the host PC. Designed as an M.2 Key B card that plugs into a laptop WWAN slot.

## Architecture

```
Laptop M.2 WWAN slot
  |
  +-- USB 2.0 (D+/D-) --> CH334R hub --> Port 1: ESP32-S3 native USB (TinyUSB HID)
  |                                  --> Port 2: CH340C (flash/debug UART)
  +-- 3.3V power -------> ESP32-S3 + NRF24L01+ + CH334R + CH340C
  +-- ~RESET (pin 67) ---> RC filter --> ESP32 EN

ESP32-S3 <--SPI--> NRF24L01+ #1 <--2.4GHz--> Left half
                   NRF24L01+ #2 <--2.4GHz--> Right half
```

## Components

| Component | Role | Package |
|-----------|------|---------|
| ESP32-S3-WROOM-2 | MCU, USB HID | Module |
| NRF24L01+ (x2) | Wireless RX from each half | Breakout |
| CH334R | USB 2.0 hub, crystal-free, 3.3V | QSOP-16 |
| CH340C | USB-to-UART for programming | SOP-16 |
| UMH3N (x2) | Auto-reset (DTR/RTS to EN/IO0) | SOT-363 |

## Form Factor

**M.2 Key B 3042** (22 x 42 mm), PCB thickness 0.8 mm. Fits laptop WWAN slots (tested: Dell Latitude 5430).

## Power

Powered directly from the M.2 slot 3.3V rail. No onboard voltage regulator -- CH334R runs at 3.3V with internal LDO bypassed (pin 12 and 13 both tied to 3.3V, confirmed by WCH datasheet).

## USB Hub (CH334R)

Both USB ports accessible without removing the dongle:
- **Port 1**: ESP32-S3 native USB (TinyUSB HID keyboard)
- **Port 2**: CH340C UART (flash and debug)
- **Ports 3-4**: unused (NC)

Crystal-free mode: XI pin (16) tied to GND.

## M.2 Key B Connections

| M.2 Pin | Signal | Connection |
|---------|--------|------------|
| 7 | USB_D+ | CH334R pin 11 (DMU+) |
| 9 | USB_D- | CH334R pin 10 (DMU-) |
| 2,4,71,73,75 | +3.3V | Power rail + C 10uF decoupling |
| 3,5,11,70,72 | GND | Ground |
| 8 | ~{W_DISABLE1} | R 10k pull-up to 3.3V |
| 6 | ~{FULL_CARD_POWER_OFF} | R 10k pull-up to 3.3V |
| 67 | ~{RESET} | R 10k + C 100nF RC filter to ESP32 EN |
| 66 | SIM_DETECT | NC |
| 12-19 | Key B notch | No pads |
| All others | NC | Not connected |

## PCB

- KiCad 9, designed for JLCPCB fabrication
- M.2 edge connector footprint (Key B)
- PCB thickness: 0.8 mm (M.2 standard)
- NRF24L01+ on bottom side, antennas extending past board edge
- All SMD except NRF24L01+ breakout headers
