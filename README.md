# KeSp_dongle

USB receiver dongles for the **KaSe** wireless split ergonomic keyboard.
Each keyboard half transmits over 2.4 GHz through an NRF24L01+; the dongle
receives both links and presents itself to the host as a USB HID keyboard.

KiCad 10 projects. Ecosystem: [KeSp_firmware](https://github.com/mornepousse/KeSp_firmware),
[KeSp_software](https://github.com/mornepousse/KeSp_software),
[KaSe_PCB](https://github.com/mornepousse/KaSe_PCB).

*[Version française plus bas](#version-française).*

| Top | Bottom |
|---|---|
| ![Board top side](images/top.png) | ![Board bottom side](images/bottom.png) |

`dongle-s3` in revision **V1.0** — the board currently in service.
Top: the CH334R (U6), the CH340C (U4), the ESP32-S3-WROOM-1 (U7) and the M.2
Key B card edge. Bottom: the two NRF24L01+ modules (U2 and U3) on their 2 mm
headers, one per keyboard half.

## The three variants

| Folder | MCU | Form factor | Status |
|---|---|---|---|
| `dongle-s3/` | ESP32-S3-WROOM-1 | M.2 Key B 3042 (22x42 mm) | **in service** — V1.0 running in the laptop's M.2 slot for several months; source now at V1.1 |
| `dongle-p4/` | ESP32-P4 QFN104 + ESP32-S3-WROOM-1 | M.2 Key B | work in progress |
| `dongle-ext/` | ESP32-S3-WROOM-1 | 2.54 mm headers, no M.2 card edge | prototype |

`dongle-s3` is the reference design and the only one proven in real use: the
V1.0 board has been sitting in the M.2 WWAN slot of a Dell Latitude 5430 for
several months, enumerating and working as a USB HID keyboard.

## `dongle-s3` architecture

**Radio** — 2x NRF24L01+ on a shared SPI bus, one per keyboard half.
Separate CSN/CE/IRQ lines, 100 Ω series resistors on every signal.

**USB** — the M.2 slot provides a single USB 2.0 link, split by a hub:

```
M.2 pins 7/9 (D+/D-)
        │
        └─ CH334R (USB 2.0 hub, QSOP-16, crystal-free mode: XI → GND)
             ├── port 1 → ESP32-S3 native USB   (HID in production)
             ├── port 2 → CH340C                (flash and debug without unplugging)
             └── ports 3-4 → unused
```

The CH334R runs natively at 3.3 V with its internal LDO bypassed: **pin 12
(5V) and pin 13 (VDD33) are both tied to 3.3 V**. Leaving pin 12 floating
makes the part overheat and prevents USB enumeration — the classic trap with
this package.

**Power** — no onboard regulator, 3.3 V comes straight from the M.2 slot
(pins 2, 4, 70, 72, 74; grounds on 3, 5, 11, 71, 73).

**M.2 control signals** — 10 kΩ pull-ups to 3.3 V on the three host control
inputs, so the host cannot disable the card at boot:

| Signal | M.2 pin | Resistor |
|---|---|---|
| `W_DISABLE1#` | 8 | R21 |
| `RESET#` | 67 | R22 |
| `FULL_CARD_POWER_OFF#` | 6 | R23 |

**Auto-reset** — 2x UMH3N driven from the CH340C DTR/RTS lines to EN and IO0.

## Libraries

Custom footprints live in `libs/` and are referenced as
`${KIPRJMOD}/../libs/…` by each project's `fp-lib-table` — no absolute paths,
so the repository clones anywhere.

| Library | Contents |
|---|---|
| `libs/MaeLid.pretty` | NRF24L01, ESP32-S3-DevKitC, JC-ESP32P4-M3, logo |
| `libs/M.2-cards` | M.2 card outline templates |
| `libs/NGFF.pretty` | M.2 / NGFF connectors |

Everything else comes from the standard KiCad libraries and from
`PCM_Espressif` (Plugin and Content Manager, install separately).

## Open items

- `dongle-s3`: both UMH3N transistors carry the same `Q1` reference. KiCad
  keeps only one of them, and only one is placed on the PCB. Re-annotate to
  Q1/Q2, then run *Update PCB from Schematic* to add the missing footprint.
- `dongle-s3`: CH334R ports 3 and 4 are labelled `D±_EXTEND1/2` in the
  schematic but not yet propagated to the PCB (4 net renames).

## Fabrication

Each variant's gerbers live in its own `Gerber/` subfolder.
The `dongle-s3` set is dated 2026-05-27 and matches the batch that was
manufactured, revision V1.0. The source has since moved to V1.1: R9 removed
and decoupling completed (C8 through C12). The gerbers are therefore no
longer in sync with the PCB.

Exported netlists (`*.net`) are not version-controlled: they are regenerated
from the schematic, and importing a stale one overwrites current work.

---

# Version française

Dongles USB récepteurs pour le clavier ergonomique sans fil **KaSe**.
Chaque moitié du clavier émet en 2,4 GHz via un NRF24L01+ ; le dongle reçoit
les deux liens et se présente au PC comme un clavier USB HID.

Projets KiCad 10. Écosystème : [KeSp_firmware](https://github.com/mornepousse/KeSp_firmware),
[KeSp_software](https://github.com/mornepousse/KeSp_software),
[KaSe_PCB](https://github.com/mornepousse/KaSe_PCB).

Les rendus plus haut montrent `dongle-s3` en révision **V1.0**, la carte
actuellement en service. Dessus : le CH334R (U6), le CH340C (U4),
l'ESP32-S3-WROOM-1 (U7) et le bord de carte M.2 Key B. Dessous : les deux
NRF24L01+ (U2 et U3) sur leurs embases 2 mm, un par moitié de clavier.

## Les trois variantes

| Dossier | MCU | Format | État |
|---|---|---|---|
| `dongle-s3/` | ESP32-S3-WROOM-1 | M.2 Key B 3042 (22x42 mm) | **en service** — V1.0 montée dans le slot M.2 du portable depuis plusieurs mois ; source en V1.1 |
| `dongle-p4/` | ESP32-P4 QFN104 + ESP32-S3-WROOM-1 | M.2 Key B | en cours |
| `dongle-ext/` | ESP32-S3-WROOM-1 | connecteurs 2,54 mm, pas de bord M.2 | prototype |

`dongle-s3` est la version de référence, et la seule éprouvée à l'usage : la
carte V1.0 est montée dans le slot M.2 WWAN d'un Dell Latitude 5430 depuis
plusieurs mois, où elle énumère et fonctionne en clavier USB HID.

## Architecture de `dongle-s3`

**Radio** — 2x NRF24L01+ sur bus SPI partagé, un par moitié de clavier.
CSN/CE/IRQ séparés, résistances série de 100 Ω sur chaque signal.

**USB** — le slot M.2 fournit un seul lien USB 2.0, dédoublé par un hub :

```
M.2 pins 7/9 (D+/D-)
        │
        └─ CH334R (hub USB 2.0, QSOP-16, mode sans quartz : XI → GND)
             ├── port 1 → USB natif de l'ESP32-S3   (HID en production)
             ├── port 2 → CH340C                    (flash et debug sans démonter)
             └── ports 3-4 → non utilisés
```

Le CH334R tourne en 3,3 V natif, LDO interne bypassé : **pin 12 (5V) et
pin 13 (VDD33) toutes deux au 3,3 V**. Laisser la pin 12 flottante fait
chauffer le composant et empêche l'énumération — c'est le piège de ce boîtier.

**Alimentation** — aucun régulateur embarqué, le 3,3 V vient directement du
slot M.2 (pins 2, 4, 70, 72, 74 ; masses sur 3, 5, 11, 71, 73).

**Signaux M.2** — pull-ups 10 kΩ vers 3,3 V sur les trois entrées de contrôle
de l'hôte, pour que la carte ne soit pas désactivée au démarrage :

| Signal | Pin M.2 | Résistance |
|---|---|---|
| `W_DISABLE1#` | 8 | R21 |
| `RESET#` | 67 | R22 |
| `FULL_CARD_POWER_OFF#` | 6 | R23 |

**Reset automatique** — 2x UMH3N sur DTR/RTS du CH340C vers EN et IO0.

## Bibliothèques

Les empreintes maison vivent dans `libs/` et sont référencées en
`${KIPRJMOD}/../libs/…` par le `fp-lib-table` de chaque projet — aucun chemin
absolu, le dépôt se clone n'importe où.

| Lib | Contenu |
|---|---|
| `libs/MaeLid.pretty` | NRF24L01, ESP32-S3-DevKitC, JC-ESP32P4-M3, logo |
| `libs/M.2-cards` | gabarits de cartes M.2 |
| `libs/NGFF.pretty` | connecteurs M.2 / NGFF |

Le reste vient des bibliothèques standard KiCad et de `PCM_Espressif`
(gestionnaire de contenu, à installer séparément).

## Points ouverts

- `dongle-s3` : les deux UMH3N portent la même référence `Q1`. KiCad n'en
  retient qu'un, et un seul est posé sur le PCB. À réannoter en Q1/Q2, puis
  *Update PCB from Schematic* pour ajouter l'empreinte manquante.
- `dongle-s3` : les ports 3 et 4 du CH334R sont étiquetés `D±_EXTEND1/2` au
  schématique mais pas encore reportés sur le PCB (4 renommages de nets).

## Fabrication

Les gerbers de chaque variante sont dans son sous-dossier `Gerber/`.
Ceux de `dongle-s3` datent du 27/05/2026 et correspondent à la série reçue, en
révision V1.0. La source a depuis évolué en V1.1 : R9 supprimée et découplage
complété (C8 à C12). Les gerbers ne sont donc plus à jour vis-à-vis du PCB.

Les netlists (`*.net`) ne sont pas versionnées : elles se régénèrent depuis le
schématique, et en importer une périmée écrase le travail en cours.
