# M.2 Key B Integration -- Reference & Sources

Documentation de reference pour l'integration du dongle KaSe v2 dans un slot M.2 WWAN.

## Specification M.2

### Pinout Key B (Socket 2)

Source: [Congatec AN43](https://wiki.congatec.com/wiki/M.2_Pinout_Descriptions_and_Reference_Designs_(AN43))

| Pin | Signal | Pin | Signal |
|-----|--------|-----|--------|
| 1 | CONFIG_3 | 2 | 3.3V |
| 3 | GND | 4 | 3.3V |
| 5 | GND | 6 | ~FULL_CARD_POWER_OFF |
| 7 | **USB_D+** | 8 | ~W_DISABLE1 |
| 9 | **USB_D-** | 10 | GPIO_9/DAS/~DSS/~LED1 |
| 11 | GND | 12-19 | **(Key B notch)** |
| 20 | GPIO_5 | 21 | CONFIG_0 |
| 22 | GPIO_6 | 23 | GPIO_11 |
| 24 | DPR | 25 | GPIO_7 |
| 26 | GND | 27 | GPIO_10 |
| 28 | GPIO_8 | 29 | PERn1/USB3.0-Rx- |
| 30 | UIM-RESET | 31 | PERp1/USB3.0-Rx+ |
| 32 | UIM-DATA | 33 | GND |
| 34 | UIM-PWR | 35 | PETn1/USB3.0-Tx- |
| 36 | UIM-CLK | 37 | PETp1/USB3.0-Tx+ |
| 38 | DEVSLP | 39 | GND |
| 40 | GPIO_0/SMB_CLK | 41 | PERn0/SATA-B+ |
| 42 | GPIO_1/SMB_DATA | 43 | PERp0/SATA-B- |
| 44 | GPIO_2/~ALERT | 45 | GND |
| 46 | GPIO_3 | 47 | PETn0/SATA-A- |
| 48 | GPIO_4 | 49 | PETp0/SATA-A+ |
| 50 | ~PERST | 51 | GND |
| 52 | ~CLKREQ | 53 | REFCLKn |
| 54 | ~PEWAKE | 55 | REFCLKp |
| 56 | GND | 57 | NC |
| 58 | ANTCTL0 | 59 | NC |
| 60 | ANTCTL1 | 61 | ANTCTL2 |
| 62 | ANTCTL3 | 63 | COEX3 |
| 64 | COEX_TXD | 65 | COEX_RXD |
| 66 | SIM_DETECT | 67 | **~RESET** |
| 68 | CONFIG_1 | 69 | SUSCLK(32kHz) |
| 70 | GND | 71 | **3.3V** |
| 72 | GND | 73 | **3.3V** |
| 74 | CONFIG_2 | 75 | **3.3V** |

Note: le symbole KiCad `Connector:Bus_M.2_Socket_B` a pin 66=SIM_DETECT et pin 67=~RESET (pas l'inverse).

### Dimensions M.2

| Format | Largeur | Longueur | Usage typique |
|--------|---------|----------|---------------|
| 2230 | 22mm | 30mm | WiFi, SSD |
| 2242 | 22mm | 42mm | SSD, WWAN |
| **3042** | **30mm** | **42mm** | **WWAN** |
| 2280 | 22mm | 80mm | SSD |

Epaisseur PCB: **0.8mm** (spec M.2 standard).

Le slot WWAN du Dell Latitude 5430 accepte du 3042 (module DW5820e = 30x42mm).

### CONFIG pins

Source: [Knightli M.2 Pinout Notes](https://www.knightli.com/en/2026/04/15/m2-pinout-descriptions/)

Les CONFIG pins sont "legacy" -- Linux n'utilise pas ces pins pour l'identification, il enumere le bus USB directement.
Source: [NVIDIA Forum](https://forums.developer.nvidia.com/t/m-2-key-b-config-x-signals-config-0-config-1-etc/335890)

Pour le dongle: tous les CONFIG en NC.

## CH334R -- Hub USB 2.0

### Datasheet

- [CH334/335 Datasheet V2.5 (PDF)](https://cdn-learn.adafruit.com/assets/assets/000/131/435/original/CH334DS1.PDF)
- [WCH Product Page](https://www.wch-ic.com/products/CH334.html)
- [CH334R sur JLCPCB (C4154405)](https://jlcpcb.com/partdetail/WCH_Jiangsu_Qin_Heng-CH334R/C4154405)

### Pinout CH334R (QSOP-16)

```
        CH334R (QSOP-16)
    +------------------------+
  1 | DM4-          XI    16 |
  2 | DM4+          XO    15 |
  3 | DM3-         GND    14 |
  4 | DM3+      VDD33    13 |
  5 | DM2-          5V    12 |
  6 | DM2+        DMU+   11 |
  7 | DM1-        DMU-   10 |
  8 | DM1+     RST/CDP    9 |
    +------------------------+
```

### Alimentation 3.3V (sans 5V)

La datasheet WCH confirme:

> "If there is an on-board 3.3V supply, both V5 and VDD33 of the HUB chip can be connected to the 3.3V supply. For industrial grade applications, it is recommended to connect both V5 and VDD33 to an external 3.3V power supply."

La variante CH334Q n'a pas de LDO interne ni de pin V5, preuve que le chip fonctionne en 3.3V natif.

Configuration dongle:
- Pin 12 (V5) -> 3.3V
- Pin 13 (VDD33) -> 3.3V + C 10uF

### Mode sans cristal

> "When the XO pin is suspended and the XI pin is connected to GND, the crystal-free mode is selected."

- Pin 16 (XI) -> GND
- Pin 15 (XO) -> NC

## Dell Latitude 5430 -- Slot WWAN

### Module d'origine

- Intel XMM 7360 (Dell DW5820e)
- LTE Cat9, 450Mbps DL
- Interface: **USB 2.0** (pas PCIe)
- Format: M.2 3042 Key B
- Source: [Intel XMM 7360 Specs](https://www.intel.com/content/www/us/en/products/sku/66649/intel-xmm-7360/specifications.html)

### Caracteristiques du slot

- USB 2.0 confirme (le DW5820e est un modem USB)
- 3.3V fourni par le slot
- PCIe present mais desactive dans le BIOS Dell (pas modifiable)
- Source: [Matt's Tech Pages - Dell M.2 WWAN](https://www.mattmillman.com/does-the-dell-latitude-m-2-wwan-socket-have-the-sata-interface-on-it/)

### BIOS Whitelist

- Dell est historiquement **permissif** sur le slot WWAN (pas de whitelist agressive comme Lenovo)
- Des utilisateurs ont installe des SSD dans le slot WWAN Dell sans probleme
- Risque residuel: le BIOS pourrait bloquer un VID/PID USB non reconnu
- Source: [Dell Community - SSD in WWAN slot](https://www.dell.com/community/en/conversations/latitude/info-installing-a-m2-ssd-in-the-wwan-slot-e5250-it-worked/647f85f2f4ccf8a8de4db82d)

### Test avant fabrication

Utiliser un adaptateur M.2 Key B -> USB en sens inverse pour verifier que le slot enumere un device USB custom.
- [Waveshare USB to M.2 B Key](https://www.waveshare.com/wiki/USB_TO_M.2_B_KEY)

## Footprints M.2 pour KiCad

### Utilises dans ce projet

- [timonsku/M.2-Card-Footprints](https://github.com/timonsku/M.2-Card-Footprints) -- Key B verifie sur PCB reel. Eagle + KiCad.
- [jmgao/kicad-ngff](https://github.com/jmgao/kicad-ngff) -- Footprints + symboles KiCad. Keys A, B, B+M, E, M.

### Autres librairies

- [realteck-ky/kicad-m2ngff](https://github.com/realteck-ky/kicad-m2ngff) -- Librairie KiCad M.2 NGFF.
- Symbole KiCad natif: `Connector:Bus_M.2_Socket_B`

## Projets similaires (detournement M.2)

- [timonsku/RP2040-M.2](https://github.com/timonsku/RP2040-M.2) -- RP2040 sur carte M.2 Key B, USB via slot WWAN. Le plus proche de notre projet.
- [enjoy-digital/litex_m2sdr](https://github.com/enjoy-digital/litex_m2sdr) -- FPGA SDR/RF sur M.2. GNU Radio compatible.
- [magic-blue-smoke/Dual-Edge-TPU-Adapter](https://github.com/magic-blue-smoke/Dual-Edge-TPU-Adapter) -- 2x Google Coral TPU sur un slot M.2.
- [themainframe/5g-m2-usb3-interface-pcb](https://github.com/themainframe/5g-m2-usb3-interface-pcb) -- Breakout USB3 M.2 Key B en KiCad.

## Articles de reference

- [M.2 For Hackers -- Connectors (Hackaday)](https://hackaday.com/2022/11/03/m-2-for-hackers-connectors/)
- [M.2 For Hackers -- Cards (Hackaday)](https://hackaday.com/2022/11/07/m-2-for-hackers-cards/)
- [M.2 For Hackers -- Expand Your Laptop (Hackaday)](https://hackaday.com/2022/10/27/m-2-for-hackers-expand-your-laptop/)
- [PCIe For Hackers -- An M.2 Card Journey (Hackaday)](https://hackaday.com/2023/07/20/pcie-for-hackers-an-m-2-card-journey/)
- [M.2 Connector Pinout (PinoutGuide)](https://pinoutguide.com/HD/M.2_NGFF_connector_pinout.shtml)
- [M.2 Pinout Generator (Nodeloop)](https://nodeloop.org/tools/m2-pinout-generator/)
