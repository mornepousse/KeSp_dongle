# Hôte ThinkPad P16 Gen 1 — le slot WWAN n'alimente pas la V1.0

Constaté le 2026-09-17 sur un ThinkPad P16 Gen 1 `21D60019FR` (Alder Lake HX,
BIOS N3FET46W 1.70, SKU sans module WWAN **mais avec les 4 antennes**).

## Symptôme

La carte `dongle-s3` V1.0, qui fonctionne dans le slot WWAN d'un Dell Latitude
5430, **n'apparaît nulle part** sur le P16 : aucune tentative d'énumération USB
dans `dmesg`, aucun `rfkill` WWAN, aucune option « Wireless WAN » au BIOS.
Autrement dit le slot n'est jamais mis sous tension.

## Cause : les pins CONFIG[3:0]

La spec M.2 fait lire à l'hôte quatre broches de configuration **avant**
d'alimenter la carte, pour savoir ce qui est inséré :

> *The system shall read all four configuration pins to identify the selected
> pinout configuration. The system shall pull-up these configuration pins to an
> appropriate power rail so the configuration pins can be read even if the M.2
> expansion card is not powered.*
> — congatec AN43, « M.2 Pinout Descriptions and Reference Designs », p. 9

C'est donc **l'hôte qui fournit les pull-ups** ; la carte se contente de tirer
à la masse ce qu'elle veut déclarer. `0` = relié à GND sur la carte, `1` =
laissé en l'air.

| CONFIG_0 (21) | CONFIG_1 (69) | CONFIG_2 (75) | CONFIG_3 (1) | Interface déclarée |
|---|---|---|---|---|
| 0 | 0 | 0 | 0 | SSD SATA |
| 0 | 1 | 0 | 0 | SSD PCIe |
| 0 | 0 | 1 | 0 | WWAN PCIe (port cfg 0) |
| 0 | 0 | 0 | 1 | WWAN PCIe + USB3.1 Gen1 (port cfg 0) |
| 1 | 0 | 0 | 0 | WWAN SSIC (port cfg 0) |
| **1** | **1** | **1** | **1** | **No Add-in Card Present** |

Sur la V1.0, **les quatre broches sont flottantes** → l'hôte lit `1111`,
comprend « pas de carte », et l'EC ne délivre jamais le 3,3 V. Le Dell 5430
alimentait le slot sans lire CONFIG : c'est lui qui était laxiste, pas le
Lenovo qui est cassé. La carte est simplement non conforme sur ce point.

## Conséquence pour le design (V1.1)

Ajouter **quatre jumpers à souder** (SJ) reliant CONFIG_0..3 à la masse, pour
pouvoir balayer les codes selon l'hôte sans refaire un PCB. Coût nul en BOM.

Ce que la doc ne permet pas de trancher, et qu'il faudra établir à l'essai :

1. Quels codes le P16 accepte d'alimenter.
2. Si les lignes **USB 2.0 (pins 7 et 9)** sont routées sur ce slot. Elles
   appartiennent au brochage WWAN — un code SSD obtiendrait peut-être
   l'alimentation mais probablement pas l'USB. Le fait que la machine ait ses
   4 antennes (donc câblée « WWAN-ready ») est un bon signe, sans garantie.

## Obstacles connus en aval

- **Whitelist BIOS ThinkPad (erreur `1802`)** : sur un code WWAN, le firmware
  vérifie l'identité de la carte — historiquement les IDs **PCI**
  vendor/device/**subsystem** plus le numéro FRU Lenovo. Un périphérique
  purement USB sera soit ignoré, soit rejeté ; inconnu à ce jour. Le P16 G1
  accepte le Fibocom L860-GL-16 et refuse le FM350 5G, donc la whitelist est
  bien active sur ce modèle.
- **Pas de BIOS modifié possible** : Alder Lake avec Intel Boot Guard, clés
  fusées dans le CPU. Les contournements de whitelist des vieux ThinkPad n'ont
  pas d'équivalent ici, même en flashant la puce SPI au clip.
- **Changer le VID/PID de l'ESP32-S3 ne sert à rien tant que le slot n'est pas
  alimenté** : sans 3,3 V il n'y a aucun descripteur USB à présenter. Et le hub
  CH334R (`1a86:8091`) reste visible avec ses IDs figés dans le silicium.
- Depuis ~2019, les SSD 2242 ne fonctionnent plus dans le slot WWAN des
  ThinkPad, même avec un BIOS modifié — cohérent avec l'hypothèse « seuls les
  codes WWAN sont alimentés ».

## Ordre d'essai recommandé

1. Multimètre, carte retirée, machine sous tension : vérifier ~3,3 V (ou 1,8 V)
   sur les pins 21, 69, 75, 1 du slot → confirme que l'hôte lit bien CONFIG.
2. Poser un code WWAN (ex. `0 0 0 1` : 21, 69, 75 à la masse, 1 flottante).
3. Vérifier l'alimentation du slot, puis `lsusb | grep 303a`.
4. Un `1802` au POST n'est pas un échec total : il prouve que le slot est
   alimenté et que le BIOS voit la carte. Bloquant mais réversible — retirer
   la carte suffit.

## Repli garanti, sans slot M.2

L'USB interne du **lecteur de carte à puce** (`2ce3:9563` sur le P16, inutilisé
ici) : connecteur carte mère n° 11 « Smart card reader connector » + nappe FFC
(FRU n° 28), dépose au § 1180 de la HMM P16 Gen 1 — après la base (§ 1070) et
le bezel clavier (§ 1150). Le lecteur tient par 2 vis M2×3,5 au bord avant
gauche, dans un logement de la taille d'un 2242. Relever VBUS / GND / D+ / D−
sur la nappe, les amener à l'amont du hub CH334R : aucune whitelist, aucun
CONFIG, alimentation garantie.

D'où une idée pour la V1.1 : prévoir un **connecteur FFC en entrée USB
alternative**, ce qui rendrait la carte utilisable sur n'importe quel portable
ayant un lecteur de carte à puce, indépendamment du slot M.2.

## Côté hôte NixOS

Les règles udev de `mae` épinglent `303a:4001` (`modules/hardware/kase-dongle.nix` :
accès hidraw non-root, symlink `/dev/ttyKASE_DONGLE`). Si le VID/PID du firmware
change un jour, ce fichier doit être mis à jour en même temps.

## Sources

- congatec AN43, *M.2 Pinout Descriptions and Reference Designs*, p. 9 (table CONFIG)
- Lenovo, *ThinkPad P16 Gen 1 Hardware Maintenance Manual* :
  `https://download.lenovo.com/pccbbs/mobiles_pdf/p16_gen1_hmm_en.pdf`
  (Table 9 connecteurs carte mère, § 1180 lecteur de carte à puce)
- ThinkWiki, *Problem with unauthorized MiniPCI network card* (mécanique de la whitelist)
