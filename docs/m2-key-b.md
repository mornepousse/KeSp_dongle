# M.2 Key B — référence pour ce projet

Matériel de référence rassemblé pour `dongle-REV1` et `dongle-REV2`. Il existait
auparavant en trois copies identiques, une par variante, y compris dans
`dongle-ext` qui n'a pas de connecteur M.2 ; elles sont fusionnées ici.

Le **résultat** de l'intégration dans un portable réel est ailleurs :
[`hote-thinkpad-p16.md`](hote-thinkpad-p16.md). Ce document-ci ne contient que
la matière première.

## Brochage Key B, socket 2

**Table régénérée depuis le symbole `Connector:Bus_M.2_Socket_B` de KiCad**,
celui sur lequel les deux cartes sont construites, et recoupée avec
[congatec AN43](https://www.congatec.com/fileadmin/user_upload/Documents/Application_Notes/AN43_M.2_Pinout_Descriptions_and_Reference_Designs.pdf)
pour les pins CONFIG.

> **Correction.** La version précédente de cette table était fausse à partir de
> la pin 68 : chaque nom y était attribué à la broche voisine, d'où un
> « pin 75 = 3,3 V » alors que la 75 porte `CONFIG_2`. L'erreur s'était
> propagée dans les trois README de variante. Les pins réellement utilisées
> sont en gras ci-dessous, et vérifiées contre la netlist des cartes.

| Pin | Signal | Pin | Signal |
|---|---|---|---|
| 1 | **CONFIG_3** | 2 | **3.3V** |
| 3 | GND | 4 | **3.3V** |
| 5 | GND | 6 | **~FULL_CARD_POWER_OFF** |
| 7 | **USB_D+** | 8 | **~W_DISABLE1** |
| 9 | **USB_D-** | 10 | GPIO_9/DAS/~DSS/~LED1 |
| 11 | GND | 12-19 | **encoche Key B — pas de pastilles** |
| 21 | **CONFIG_0** | 22 | GPIO_6 |
| 23 | GPIO_11 | 24 | GPIO_7 |
| 25 | DPR | 26 | GPIO_10 |
| 27 | GND | 28 | GPIO_8 |
| 29 | PERn1/USB3.0-Rx-/SSIC-RxN | 30 | UIM-RESET |
| 31 | PERp1/USB3.0-Rx+/SSIC-RxP | 32 | UIM-CLK |
| 33 | GND | 34 | UIM-DATA |
| 35 | PETn1/USB3.0-Tx-/SSIC-TxN | 36 | UIM-PWR |
| 37 | PETp1/USB3.0-Tx+/SSIC-TxP | 38 | DEVSLP |
| 39 | GND | 40 | GPIO_0 |
| 41 | PERn0/SATA-B+ | 42 | GPIO_1 |
| 43 | PERp0/SATA-B- | 44 | GPIO_2 |
| 45 | GND | 46 | GPIO_3 |
| 47 | PETn0/SATA-A- | 48 | GPIO_4 |
| 49 | PETp0/SATA-A+ | 50 | ~PERST |
| 51 | GND | 52 | ~CLKREQ |
| 53 | REFCLKn | 54 | ~PEWAKE |
| 55 | REFCLKp | 56 | NC |
| 57 | GND | 58 | NC |
| 59 | ANTCTL0 | 60 | COEX3 |
| 61 | ANTCTL1 | 62 | COEX2 |
| 63 | ANTCTL2 | 64 | COEX1 |
| 65 | ANTCTL3 | 66 | SIM_DETECT |
| 67 | **~RESET** | 68 | SUSCLK |
| 69 | **CONFIG_1** | 70 | **3.3V** |
| 71 | GND | 72 | **3.3V** |
| 73 | GND | 74 | **3.3V** |
| 75 | **CONFIG_2** |  |  |
Alimentation : **2, 4, 70, 72, 74**. Masses : **3, 5, 11, 27, 33, 39, 45, 51,
57, 71, 73**. C'est ce que câblent les deux cartes, netlist à l'appui.

## Formats et épaisseur

| Format | Largeur | Longueur | Usage typique |
|---|---|---|---|
| 2230 | 22 mm | 30 mm | WiFi, SSD |
| 2242 | 22 mm | 42 mm | SSD, WWAN |
| **3042** | **30 mm** | **42 mm** | **WWAN — le format de ce projet** |
| 2280 | 22 mm | 80 mm | SSD |

Épaisseur PCB : **0,8 mm**, standard M.2.

Le slot WWAN du Dell Latitude 5430 accepte du 3042 — son module d'origine, le
DW5820e, fait 30 × 42 mm.

## Les pins CONFIG — ce qu'il ne faut pas croire

Cette section disait : *« les CONFIG pins sont legacy, Linux n'utilise pas ces
pins pour l'identification, il énumère le bus USB directement. Pour le dongle :
tous les CONFIG en NC. »*

**C'est faux, et ça a coûté plusieurs semaines.** Linux n'a rien à voir
là-dedans : ce sont le BIOS et l'EC qui lisent ces broches, **avant** de mettre
le slot sous tension, donc bien avant qu'un système d'exploitation existe. Un
hôte laxiste comme le Dell alimente sans lire ; un hôte strict comme le
ThinkPad P16 Gen 1 lit, conclut « pas de carte », et ne délivre jamais le
3,3 V.

Le détail, le balayage des codes et ce qu'il établit : [`hote-thinkpad-p16.md`](hote-thinkpad-p16.md).

## CH334R — hub USB 2.0

- [Datasheet CH334/335 V2.5](https://cdn-learn.adafruit.com/assets/assets/000/131/435/original/CH334DS1.PDF)
- [Page produit WCH](https://www.wch-ic.com/products/CH334.html)
- [CH334R chez JLCPCB (C4154405)](https://jlcpcb.com/partdetail/WCH_Jiangsu_Qin_Heng-CH334R/C4154405)

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

**Fonctionnement en 3,3 V sans 5 V.** La datasheet WCH l'autorise explicitement :

> *If there is an on-board 3.3V supply, both V5 and VDD33 of the HUB chip can be
> connected to the 3.3V supply. For industrial grade applications, it is
> recommended to connect both V5 and VDD33 to an external 3.3V power supply.*

La variante CH334Q n'a ni LDO interne ni pin V5, ce qui confirme que la puce est
nativement 3,3 V. Configuration retenue : pin 12 (V5) et pin 13 (VDD33) toutes
deux au 3,3 V, avec 10 µF de découplage. **Laisser la pin 12 en l'air fait
chauffer le composant et empêche l'énumération** — c'est le piège de ce boîtier.

**Mode sans quartz** : pin 16 (XI) à la masse, pin 15 (XO) en l'air.

> *When the XO pin is suspended and the XI pin is connected to GND, the
> crystal-free mode is selected.*

Le hub a été **supprimé en REV2** : l'ESP32-S3 attaque directement les pins 7/9
du M.2. Cette section reste pour `dongle-REV1` et `dongle-ext`, qui le portent
encore.

## Dell Latitude 5430 — le slot qui marche

- Module d'origine : Intel XMM 7360 (Dell DW5820e), LTE Cat9, **interface USB 2.0**, M.2 3042 Key B
- USB 2.0 confirmé par le fait même que le module d'origine est un modem USB
- 3,3 V fourni par le slot
- PCIe présent mais désactivé dans le BIOS Dell, non modifiable
- Dell est historiquement permissif sur ce slot, contrairement à Lenovo : des
  utilisateurs y ont installé des SSD sans difficulté
- Et surtout : **il alimente sans lire les pins CONFIG**, ce qui explique que la
  V1.0 y fonctionne malgré ses quatre broches flottantes

Sources : [Intel XMM 7360](https://www.intel.com/content/www/us/en/products/sku/66649/intel-xmm-7360/specifications.html),
[Matt's Tech Pages](https://www.mattmillman.com/does-the-dell-latitude-m-2-wwan-socket-have-the-sata-interface-on-it/),
[Dell Community](https://www.dell.com/community/en/conversations/latitude/info-installing-a-m2-ssd-in-the-wwan-slot-e5250-it-worked/647f85f2f4ccf8a8de4db82d).

## Empreintes M.2 pour KiCad

Utilisées ici :

- [timonsku/M.2-Card-Footprints](https://github.com/timonsku/M.2-Card-Footprints) — Key B vérifié sur PCB réel, Eagle et KiCad
- [jmgao/kicad-ngff](https://github.com/jmgao/kicad-ngff) — empreintes et symboles, keys A, B, B+M, E, M
- Symbole natif KiCad : `Connector:Bus_M.2_Socket_B`

Autre piste : [realteck-ky/kicad-m2ngff](https://github.com/realteck-ky/kicad-m2ngff).

## Projets similaires — détourner un slot M.2

- [timonsku/RP2040-M.2](https://github.com/timonsku/RP2040-M.2) — RP2040 sur carte M.2 Key B, USB par le slot WWAN. Le plus proche de ce projet, et il laisse lui aussi ses quatre CONFIG en l'air.
- [enjoy-digital/litex_m2sdr](https://github.com/enjoy-digital/litex_m2sdr) — FPGA SDR sur M.2
- [magic-blue-smoke/Dual-Edge-TPU-Adapter](https://github.com/magic-blue-smoke/Dual-Edge-TPU-Adapter) — deux Coral TPU sur un slot M.2
- [themainframe/5g-m2-usb3-interface-pcb](https://github.com/themainframe/5g-m2-usb3-interface-pcb) — breakout USB3 Key B en KiCad

## Articles

- [M.2 For Hackers — Expand Your Laptop](https://hackaday.com/2022/10/27/m-2-for-hackers-expand-your-laptop/)
- [M.2 For Hackers — Connectors](https://hackaday.com/2022/11/03/m-2-for-hackers-connectors/)
- [M.2 For Hackers — Cards](https://hackaday.com/2022/11/07/m-2-for-hackers-cards/)
- [PCIe For Hackers — An M.2 Card Journey](https://hackaday.com/2023/07/20/pcie-for-hackers-an-m-2-card-journey/)
- [M.2 Connector Pinout (PinoutGuide)](https://pinoutguide.com/HD/M.2_NGFF_connector_pinout.shtml)
