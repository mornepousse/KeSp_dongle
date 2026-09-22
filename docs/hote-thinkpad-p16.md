# Hôte ThinkPad P16 Gen 1 — le slot WWAN n'alimente pas la V1.0

Constaté le 2026-09-17 sur un ThinkPad P16 Gen 1 `21D60019FR` (Alder Lake HX,
BIOS N3FET46W 1.70, SKU sans module WWAN **mais avec les 4 antennes**).

Mis à jour le 2026-09-22 : retours publics rassemblés, correction de ce qui
était écrit ici sur la whitelist, et état de la carte `dongle-REV2`.

## Symptôme

La carte `dongle-REV1` en V1.0 — l'ancien `dongle-s3` —, qui fonctionne dans le
slot WWAN d'un Dell Latitude 5430, **n'apparaît nulle part** sur le P16 : aucune
tentative d'énumération USB dans `dmesg`, aucun `rfkill` WWAN, aucune option
« Wireless WAN » au BIOS. Autrement dit le slot n'est jamais mis sous tension.

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

## Ce que font les autres cartes USB en slot M.2

L'omission n'est pas propre à cette carte. **`timonsku/RP2040-M.2`**, la
référence open source d'une carte M.2 B-key purement USB 2.0, câble 30 pins sur
75 et **laisse ses quatre CONFIG en l'air** — vérifié en cartographiant son
schéma : pins 1, 21, 69 et 75 non connectées, `RESET#` (67) non plus, seuls
`FULL_CARD_POWER_OFF#` et `W_DISABLE1#` sont repris. Elle déclare donc `1111`
elle aussi. Tout le monde teste sur des hôtes laxistes, et personne n'a eu à
résoudre le cas strict : d'où l'absence de documentation sur le sujet.

La catégorie existe pourtant en produit fini : des cartes M.2 Key B qui vivent
dans le slot WWAN et y présentent un périphérique USB — lecteurs microSD, et
des adaptateurs dont la notice liste explicitement « USB wireless cards, **USB
mouse receivers**, USB SSDs », avec une limite de 28 mm de long.

Côté ThinkPad précisément, le modem Sierra EM7455 « is connected internally by
USB in ThinkPad devices » : ces machines câblent et énumèrent bien l'USB 2.0 de
leur slot WWAN. Et la série M.2 de Hackaday résume la situation des slots B :
« *you can only really rely on USB 2.0 being present – everything else highly
varies* ». Le seul bus fiable dans un slot B est précisément celui dont cette
carte a besoin.

**Ce qu'on n'a pas trouvé** : aucun retour portant sur un P16 Gen 1, et aucun
cas documenté d'un Lenovo refusant d'alimenter une carte à cause de
`CONFIG = 1111`. L'hypothèse de ce document est la meilleure explication
disponible, cohérente avec tout ce qui précède, mais elle n'est corroborée par
aucun témoignage direct.

## Les deux verrous, et leur poids réel

### 1. L'alimentation du slot — le vrai obstacle

C'est ici que tout se joue, et la recherche l'a plutôt durci. La carte que le
P16 accepte, la Fibocom L860-GL, a pour interface de données le **PCIe Gen2**,
l'USB2 n'y servant qu'au debug. Le slot du P16 est donc provisionné pour du
PCIe. Que les paires USB 2.0 des pins 7/9 soient effectivement câblées reste
probable — elles appartiennent au brochage Key B WWAN et la L860 s'en sert —
mais ce n'est pas démontré sur ce modèle.

Reste une question que nulle datasheet ne tranche : si le BIOS lit un code
CONFIG WWAN, cherche un périphérique PCIe et n'en trouve aucun, laisse-t-il le
rail 3,3 V debout ?

### 2. La whitelist 1802 — moins menaçante qu'annoncé

**Correction de ce qui était écrit ici.** Le contrôle 1802 vérifie le
**sub-vendor PCI-ID** de la carte, et pour le WWAN il exige en plus un numéro
FRU Lenovo listé. Il mord donc sur un identifiant **PCI**. Une carte purement
USB n'a pas de PCI-ID du tout : elle n'existe pas dans l'espace de configuration
PCI, et le test n'a rien à mordre.

C'est cohérent avec le fait que des modems USB fonctionnent dans ces slots :
quand un EM7455 se fait refuser, c'est par son PCI-ID de sous-système, pas
parce qu'il est USB.

Ce verrou-là n'est donc probablement pas celui qui bloque. À noter tout de
même : **aucun BIOS modifié n'est possible** sur cette machine — Alder Lake
avec Intel Boot Guard, clés fusées dans le CPU. Les contournements des vieux
ThinkPad n'ont pas d'équivalent, même en flashant la puce SPI au clip.

Et rappel utile : **changer le VID/PID de l'ESP32-S3 ne sert à rien tant que le
slot n'est pas alimenté**. Sans 3,3 V il n'y a aucun descripteur USB à
présenter.

## Protocole d'essai, sur une V1.0 existante

Rien de tout ceci ne justifie un nouveau PCB avant d'avoir mesuré. Les pins
CONFIG sont des doigts dorés accessibles : un fil fin entre la pin à déclarer et
une masse voisine (3, 5 ou 11) suffit à poser un code sur une carte déjà en
main. Une soirée, zéro fab.

1. Multimètre, carte retirée, machine sous tension : vérifier ~3,3 V (ou 1,8 V)
   sur les pins 21, 69, 75, 1 du slot → confirme que l'hôte lit bien CONFIG.
2. Poser un code WWAN au fil (ex. `0 0 0 1` : 21, 69, 75 à la masse, 1
   flottante), puis balayer les autres codes WWAN du tableau.
3. Vérifier l'alimentation du slot, puis `lsusb | grep 303a`.
4. Un `1802` au POST n'est pas un échec total : il prouve que le slot est
   alimenté et que le BIOS voit la carte. Bloquant mais réversible — retirer
   la carte suffit.

Si le slot reste mort quel que soit le code, aucun respin n'y changera rien et
c'est le repli FFC ci-dessous qui devient la réponse.

## État de la carte REV2

`dongle-REV2` intègre les conclusions ci-dessus. Par rapport à REV1 :

| Modification | Raison |
|---|---|
| `JP2`–`JP5` : CONFIG_0..3 vers GND, **ouverts** par défaut | Permet de balayer les codes selon l'hôte sans refaire un PCB. Coût nul en BOM. |
| Hub CH334R (`U6`) supprimé, avec `C6` et `J3` | L'hôte ne voit plus qu'un périphérique, dont le firmware maîtrise le VID/PID. Supprime aussi le piège du 3,3 V natif de ce boîtier. |
| ESP32-S3 (`U7:13/14`) directement sur `J2:9/7` | USB 2.0 du M.2 attaqué sans intermédiaire. |
| CH340C (`U4`) déplacé sur `J1` (1 GND, 2 D−, 3 D+, 4 VBUS) | Console série et auto-reset conservés, en port de programmation de banc. |
| `U1` LD1117S33 + `C13`/`C14`, précédé de `D1` (SS14) | Alimente la carte depuis le VBUS du câble de banc. Le SS14 en série bloque le retour du 3,3 V du slot vers le connecteur par le parasite interne du régulateur. |
| `JP6` entre la sortie du LDO et le rail `+3.3V`, **ouvert** par défaut | Empêche le LDO d'alimenter le rail WWAN du portable si le câble de banc est branché carte insérée — ce qui fausserait la mesure de l'étape 1 ci-dessus. `C13` reste côté LDO, qui garde sa capacité de sortie jumper ouvert. |

Le `flash sans démonter` que permettait le CH340C derrière le hub passe
désormais par l'USB Serial/JTAG de l'ESP32-S3 sur la liaison M.2 : son CDC-ACM
« supports host controllable chip reset and entry into download mode »
(ESP32-S3 TRM v1.8, p. 1244). À noter que le PHY USB interne est **partagé**
entre l'USB-OTG et l'USB Serial/JTAG, un seul à la fois (p. 1244-1245) : quand
le firmware tient le PHY en OTG pour le HID, la console série disparaît.

### Vérification au 2026-09-22

`kicad-cli`, schéma et PCB : **0 problème de parité, 0 élément non connecté**,
41 composants de chaque côté.

- **ERC — 54 violations** : 51 sur les pins inutilisées du connecteur M.2
  (PCIe, SATA, SIM/UIM, USB 3.0), bruit normal. Restent le MISO partagé entre
  les deux nRF24 (faux positif, bus SPI) et deux `PWR_FLAG` manquants.
- **DRC — 102 violations**, dont **58 entre pads voisins du connecteur M.2
  lui-même**, toutes à 0,150 mm contre une règle à 0,200 mm : c'est le pas
  normalisé des doigts dorés, pas un défaut. Sans exception d'isolation posée
  sur `J2`, le DRC reste rouge en permanence et cesse d'être utile.
- Reste à nettoyer : 2 isolations réelles (`C9`/via `/IO0` à 0,175 mm, cuivre
  GND contre `J2:9` à 0,150 mm), une via orpheline sur le net VBUS, une piste
  orpheline de 14 µm sur `/D-_ESP32`, deux freins thermiques à un seul rayon
  (`C13`, `J1`), `J2` dans une zone interdite, et la sérigraphie.

### Points connus, non bloquants

- `C8` à `C14` portent `Resistor_SMD:R_1206_3216Metric` : même pastille, mais la
  BOM et les fichiers JLCPCB décriront une empreinte de résistance pour des
  condensateurs de 10 µF.
- `C14` est à 10 µF sur VBUS, exactement la limite admise côté périphérique par
  l'USB 2.0 ; 4,7 µF serait plus sûr à la connexion à chaud.
- Budget de tension du chemin de banc : `VBUS − 0,3 V (D1) − ~1 V (dropout du
  LD1117)`, soit 3,7 V depuis un VBUS à 5 V — confortable. Depuis un VBUS
  affaissé à 4,4 V en bout de câble il ne reste que 3,1 V et la régulation
  décroche. Une diode supplémentaire en sortie du LDO rendrait ce budget
  intenable : c'est pourquoi l'isolation de sortie est un jumper, pas une diode.
- `J1` est câblé 1 = GND, 2 = D−, 3 = D+, 4 = VBUS, soit l'inverse de l'ordre
  d'un câble USB-A. Le pigtail doit être fait en conséquence.
- Les deux symboles `Q1` sont les **unités 1 et 2 d'un même UMH3N** — un double
  transistor en SOT-363. Une seule empreinte au PCB est correcte ; le point
  ouvert du README qui demande de réannoter en Q1/Q2 est erroné.

## Repli garanti, sans slot M.2

L'USB interne du **lecteur de carte à puce** (`2ce3:9563` sur le P16, inutilisé
ici) : connecteur carte mère n° 11 « Smart card reader connector » + nappe FFC
(FRU n° 28), dépose au § 1180 de la HMM P16 Gen 1 — après la base (§ 1070) et
le bezel clavier (§ 1150). Le lecteur tient par 2 vis M2×3,5 au bord avant
gauche, dans un logement de la taille d'un 2242. Relever VBUS / GND / D+ / D−
sur la nappe et les amener à l'entrée USB de la carte : aucune whitelist, aucun
CONFIG, alimentation garantie.

D'où une idée pour une révision ultérieure : prévoir un **connecteur FFC en
entrée USB alternative**, ce qui rendrait la carte utilisable sur n'importe quel
portable ayant un lecteur de carte à puce, indépendamment du slot M.2. La place
libérée par le CH334R en REV2 y suffirait, si le LD1117 en SOT-223 n'en
occupait pas déjà l'essentiel.

## Côté hôte NixOS

Les règles udev de `mae` épinglent `303a:4001` (`modules/hardware/kase-dongle.nix` :
accès hidraw non-root, symlink `/dev/ttyKASE_DONGLE`). Si le VID/PID du firmware
change un jour, ce fichier doit être mis à jour en même temps.

## Sources

- congatec AN43, *M.2 Pinout Descriptions and Reference Designs*, p. 9 (table CONFIG) —
  `https://www.congatec.com/fileadmin/user_upload/Documents/Application_Notes/AN43_M.2_Pinout_Descriptions_and_Reference_Designs.pdf`
- Espressif, *ESP32-S3 Technical Reference Manual* v1.8, p. 1244-1245 (PHY USB
  partagé, reset et download mode par CDC-ACM)
- STMicroelectronics, *LD1117 datasheet* (dropout 1 V typ., Id 5 mA typ. /
  10 mA max) — `https://www.st.com/resource/en/datasheet/ld1117.pdf`
- `timonsku/RP2040-M.2`, carte M.2 B-key USB 2.0 open source —
  `https://github.com/timonsku/RP2040-M.2`
- Hackaday, *M.2 For Hackers – Expand Your Laptop* —
  `https://hackaday.com/2022/10/27/m-2-for-hackers-expand-your-laptop/`
- `danielewood/sierra-wireless-modems` (EM7455 relié en USB interne sur ThinkPad) —
  `https://github.com/danielewood/sierra-wireless-modems`
- Fibocom, *L860-GL Hardware User Manual*, p. 29 (interface PCIe Gen2, USB2 en debug)
- ThinkWiki, *Problem with unauthorized MiniPCI network card* (mécanique de la
  whitelist : sub-vendor PCI-ID et FRU) —
  `https://www.thinkwiki.org/wiki/Problem_with_unauthorized_MiniPCI_network_card`
- Lenovo, *ThinkPad P16 Gen 1 Hardware Maintenance Manual* —
  `https://download.lenovo.com/pccbbs/mobiles_pdf/p16_gen1_hmm_en.pdf`
  (Table 9 connecteurs carte mère, § 1180 lecteur de carte à puce)
