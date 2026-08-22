# KeSp_dongle

Dongles USB récepteurs pour le clavier ergonomique sans fil **KaSe**.
Chaque moitié du clavier émet en 2,4 GHz via un NRF24L01+ ; le dongle reçoit
les deux liens et se présente au PC comme un clavier USB HID.

Projets KiCad 10. Écosystème : [KeSp_firmware](https://github.com/mornepousse/KeSp_firmware),
[KeSp_software](https://github.com/mornepousse/KeSp_software),
[KaSe_PCB](https://github.com/mornepousse/KaSe_PCB).

## Les trois variantes

| Dossier | MCU | Format | État |
|---|---|---|---|
| `dongle-s3/` | ESP32-S3-WROOM-1 | M.2 Key B 3042 (22×42 mm) | fabriqué (gerbers du 27/05/2026) |
| `dongle-p4/` | ESP32-P4 QFN104 + ESP32-S3-WROOM-1 | M.2 Key B | en cours |
| `dongle-ext/` | ESP32-S3-WROOM-1 | connecteurs 2,54 mm, pas de bord M.2 | prototype |

`dongle-s3` est la version de référence : elle se branche dans le slot WWAN
M.2 d'un portable (validé sur Dell Latitude 5430).

## Architecture de `dongle-s3`

**Radio** — 2× NRF24L01+ sur bus SPI partagé, un par moitié de clavier.
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

**Reset automatique** — 2× UMH3N sur DTR/RTS du CH340C vers EN et IO0.

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
Ceux de `dongle-s3` datent du 27/05/2026 et correspondent à la série reçue.

---

Les netlists (`*.net`) ne sont pas versionnées : elles se régénèrent depuis le
schématique, et en importer une périmée écrase le travail en cours.
