# Hôte ThinkPad P16 Gen 1 — le slot WWAN est inerte, courant comme données

Enquête close le **2026-10-13** sur un ThinkPad P16 Gen 1 `21D60019FR`
(Alder Lake HX, BIOS N3FET46W 1.70, SKU sans module WWAN **mais avec les 4
antennes**).

**Conclusion : le slot est inerte de bout en bout.** Aucun code CONFIG ne le
fait alimenter, et en apportant le courant soi-même on ne trouve rien derrière :
le port USB qui y mène est désactivé lui aussi. Ce n'est pas une question de
trouver le bon code ; il n'y en a pas. La suite du projet passe par le lecteur
de carte à puce, pas par le M.2.

## Le relevé

Dix des seize combinaisons ont été essayées, carte insérée, machine démarrée à
chaque fois, 3,3 V du slot mesuré au multimètre. **Les six codes définis par la
spec y sont tous**, plus quatre codes non définis.

| Code `C0 C1 C2 C3` | Déclare | Au POST | 3,3 V sur le slot |
|---|---|---|---|
| `1111` | pas de carte | rien, carte invisible | non |
| `0000` | SSD SATA | rien, aucun message | non |
| `0100` | SSD PCIe | essayé | non |
| `0010` | WWAN PCIe | **1802, machine haltée** | non |
| `0001` | WWAN PCIe + USB3.1 Gen1 | **1802, machine haltée** | non |
| `1000` | WWAN SSIC | **1802, machine haltée** | non |
| `0111`, `1011`, `1101`, `1110` | non définis | essayés | non |

```
1802: Unauthorized network card is plugged in - Power off and remove the
network card.
System is halted
```

Le POST s'arrêtait en 1802 à la quasi-totalité des essais ; l'issue n'a pas été
notée code par code pour `0100` ni pour les quatre codes non définis. Ce qui est
certain et vaut pour les dix : **le rail 3,3 V n'est jamais monté.**

Non essayés : les six codes de poids 2 — `0011`, `0101`, `0110`, `1001`,
`1010`, `1100`. Aucun n'est défini dans la table congatec.

## Le mécanisme que ça établit

L'enchaînement côté hôte est : **lire CONFIG → autoriser → alimenter**. Il ne
franchit jamais la deuxième étape.

- Le comportement **change avec le code**, ce qui prouve directement que l'hôte
  lit les quatre broches **avant** de décider quoi que ce soit. C'était
  l'hypothèse de départ de ce document ; elle est confirmée par l'expérience,
  et à notre connaissance ce relevé n'existait nulle part publiquement.
- Sur un **code WWAN**, le BIOS engage le chemin WWAN, cherche à autoriser le
  module, n'y parvient pas, et halte. Le verdict par défaut d'une carte
  déclarée mais non identifiable est le rejet.
- Sur un **code non-WWAN**, il ignore la carte sans rien dire et sans
  l'alimenter : ce slot n'est pas provisionné pour autre chose que du WWAN.
  Cohérent avec le fait que les SSD 2242 ne fonctionnent plus dans les slots
  WWAN des ThinkPad depuis ~2019.
- Le rail 3,3 V n'est donc jamais mis sous tension **que pour un module WWAN
  autorisé**. Il n'existe pas de code qui obtienne le courant sans déclencher
  le contrôle d'autorisation.

Les six combinaisons non essayées sont toutes indéfinies. Par déduction du
mécanisme ci-dessus, elles tombent dans le cas « ignoré sans alimentation » —
mais c'est une déduction, pas un relevé.

## Second essai : apporter le courant soi-même

Le BIOS refuse d'alimenter. Restait l'hypothèse que les **données** soient
quand même câblées : les paires D+/D− vont typiquement du contrôleur hôte au
connecteur en direct, et c'est le rail 3,3 V qu'un load switch coupe, pas le
lien USB.

Protocole : les quatre jumpers CONFIG ouverts, donc `1111`, donc aucun 1802 et
un démarrage normal. Machine démarrée, puis 3,3 V injectés sur la carte par les
embases 2 mm des nRF24 (`U2:1` et `U2:2`, les plus gros points de soudure de la
carte), alimentation de labo à limitation de courant, `dmesg -w` ouvert.

**Résultat : aucune énumération.** Pas même le hub CH334R, qui est pourtant
indifférent à tout ce qui se passe en aval sur la carte.

Donc le BIOS ne se contente pas de couper le rail : il **désactive aussi le
port USB** qui mène au connecteur. Sans carte autorisée, le slot ne porte ni
courant ni données.

Deux variantes n'ont pas été essayées, et aucune ne paraît valoir le
déplacement : établir le 3,3 V dès le POST plutôt qu'après démarrage — si la
coupure du port découle de la lecture CONFIG, la présence d'un périphérique n'y
change rien — et relire le menu `Security → I/O Port Access` du BIOS, qui
coûte deux minutes si le sujet est rouvert un jour.

## Ce que ce résultat invalide

**La version précédente de ce document se trompait sur le 1802.** Elle
affirmait que le contrôle mordait sur le sub-vendor PCI-ID, et qu'une carte
purement USB, dépourvue de PCI-ID, ne lui offrait rien à examiner — donc qu'il
ne se déclencherait probablement pas. Il se déclenche. Le contrôle ne compare
pas un identifiant erroné à une liste : il refuse ce qu'il ne sait pas
identifier.

Tombent avec cette correction :

- **Usurper un VID/PID USB ne sert à rien.** Le BIOS halte *avant* d'alimenter
  la carte, qui n'énumère donc jamais : aucun descripteur USB ne lui parvient,
  quel qu'il soit. La piste `8087:095a` (Fibocom L860-GL) est close sans avoir
  à être essayée.
- **Toutes les pistes côté firmware** tombent de même, puisque rien sur la
  carte ne démarre.
- **Le BIOS modifié** restait déjà hors de portée : Alder Lake avec Intel Boot
  Guard, clés fusées dans le CPU, aucun équivalent aux contournements des
  vieux ThinkPad même en flashant la puce SPI au clip.

La route M.2 est **fermée** sur cette machine. Pas difficile : fermée.

## Référence — les pins CONFIG[3:0]

Conservé ici parce que le mécanisme reste valable pour tout autre hôte.

> *The system shall read all four configuration pins to identify the selected
> pinout configuration. The system shall pull-up these configuration pins to an
> appropriate power rail so the configuration pins can be read even if the M.2
> expansion card is not powered.*
> — congatec AN43, « M.2 Pinout Descriptions and Reference Designs », p. 9

C'est **l'hôte qui fournit les pull-ups** ; la carte se contente de tirer à la
masse ce qu'elle veut déclarer. `0` = relié à GND sur la carte (jumper ponté),
`1` = laissé en l'air.

| CONFIG_0 (21) | CONFIG_1 (69) | CONFIG_2 (75) | CONFIG_3 (1) | Interface déclarée |
|---|---|---|---|---|
| 0 | 0 | 0 | 0 | SSD SATA |
| 0 | 1 | 0 | 0 | SSD PCIe |
| 0 | 0 | 1 | 0 | WWAN PCIe (port cfg 0) |
| 0 | 0 | 0 | 1 | WWAN PCIe + USB3.1 Gen1 (port cfg 0) |
| 1 | 0 | 0 | 0 | WWAN SSIC (port cfg 0) |
| **1** | **1** | **1** | **1** | **No Add-in Card Present** |

Sur `dongle-REV2`, les jumpers `JP5`, `JP2`, `JP3`, `JP4` posent respectivement
CONFIG_3, _0, _1 et _2. Ils restent utiles : le Dell Latitude 5430 alimente son
slot sans lire CONFIG, mais un autre hôte strict pourrait, lui, accepter une
déclaration.

## Référence — ce que font les autres cartes USB en slot M.2

`timonsku/RP2040-M.2`, la référence open source d'une carte M.2 B-key purement
USB 2.0, câble 30 pins sur 75 et **laisse ses quatre CONFIG en l'air** —
vérifié en cartographiant son schéma. Elle déclare donc `1111` elle aussi, et
serait tout aussi invisible ici.

La catégorie existe pourtant en produit fini : des cartes M.2 Key B qui vivent
dans le slot WWAN et y présentent un périphérique USB — lecteurs microSD, et
des adaptateurs dont la notice liste « USB wireless cards, **USB mouse
receivers**, USB SSDs ». Elles fonctionnent sur les hôtes laxistes.

Côté ThinkPad, le modem Sierra EM7455 « is connected internally by USB in
ThinkPad devices » : ces machines câblent bien l'USB 2.0 de leur slot WWAN. Et
la série M.2 de Hackaday résume les slots B : « *you can only really rely on
USB 2.0 being present – everything else highly varies* ». Le bus était bien là.
C'est l'autorisation qui manque, pas le câblage.

La carte que le P16 accepte, la Fibocom L860-GL, est par ailleurs **PCIe
Gen2**, l'USB2 n'y servant qu'au debug.

## La suite : remplacer le lecteur de carte à puce

L'USB interne du **lecteur de carte à puce** (`2ce3:9563` sur le P16, inutilisé
ici) : connecteur carte mère n° 11 « Smart card reader connector » + nappe FFC
(FRU n° 28), dépose au § 1180 de la HMM P16 Gen 1 — après la base (§ 1070) et
le bezel clavier (§ 1150). Le lecteur tient par 2 vis M2×3,5 au bord avant
gauche, dans un logement de la taille d'un 2242.

**Ne pas dériver la nappe — remplacer ce qu'il y a au bout.** Une carte portant
le même connecteur FFC prend la place du lecteur dans son logement et se branche
sur la nappe d'origine. Pas de soudure au pas de 0,5 mm, pas de fil volant dans
le portable, et un montage réversible : le lecteur se remet quand on veut.

À relever à l'ouverture, avant de dessiner quoi que ce soit :

| À déterminer | Pourquoi |
|---|---|
| Pas du connecteur, nombre de contacts, type d'actionneur | donne la référence de l'empreinte |
| Sens des contacts de la nappe — même face ou face opposée | s'y tromper inverse tout le brochage |
| Quel contact porte VBUS, GND, D+, D− | **au multimètre, lecteur branché** ; une inversion de D+/D− ne se voit qu'à l'énumération |
| Cotes du logement et position des deux trous M2 | contour de la carte |

L'électronique, elle, existe déjà : `dongle-ext` porte un ESP32-S3, deux nRF24
et un LD1117S33 alimenté par le 5 V de son `J1`, câblé `1 GND, 2 D−, 3 D+,
4 VBUS` — exactement ce que porte la nappe. C'est un changement de connecteur
et de contour, pas une conception neuve. À noter que `dongle-ext` mesure
**29,90 × 35,50 mm**, à confronter aux cotes réelles du logement.

**Vérifier d'abord que `Smart Card Slot` est sur *Enabled*** dans
`Security → I/O Port Access` du BIOS. Cette machine a montré qu'elle désactive
volontiers ce qu'elle n'attend pas, et toute cette route repose sur ce port.

Piste secondaire non vérifiée : le slot M.2 Key E du WLAN porte lui aussi de
l'USB 2.0, puisque c'est par là que passe le Bluetooth. C'est un bord de carte,
donc un PCB à faire plutôt qu'un connecteur à trouver. L'occuper coûterait le
WiFi, et rien ne dit que ce slot n'a pas son propre contrôle d'autorisation.

Et si rien de tout cela n'aboutit, reste la solution que tout le monde emploie :
un dongle sur un port USB externe. Défaite esthétique, pas fonctionnelle.

## État de la carte REV2

`dongle-REV2` reste une carte valide — pour le Dell, et pour tout hôte qui
alimente son slot. Par rapport à REV1 :

| Modification | Raison |
|---|---|
| `JP2`–`JP5` : CONFIG_0..3 vers GND, **ouverts** par défaut | Déclarer un code à la goutte de soudure, et les balayer selon l'hôte, sans refaire un PCB |
| Hub CH334R (`U6`) supprimé, avec `C6` et `J3` | L'hôte ne voit plus qu'un périphérique, dont le firmware maîtrise le VID/PID ; supprime aussi le piège du 3,3 V natif de ce boîtier |
| ESP32-S3 (`U7:13/14`) directement sur `J2:9/7` | Le lien USB 2.0 du M.2 sans rien entre les deux |
| CH340C (`U4`) déplacé sur `J1` | Console série et auto-reset, en port de programmation de banc |
| `U1` LD1117S33 + `C13`/`C14`, précédé de `D1` (SS14) | Alimente la carte depuis le VBUS de `J1` ; le Schottky bloque le retour vers le connecteur |
| `JP6`, **ouvert** par défaut | Isole la sortie du LDO du rail 3,3 V du slot |

Le « flash sans démonter » passe par l'USB Serial/JTAG de l'ESP32-S3 sur la
liaison M.2 : son CDC-ACM « supports host controllable chip reset and entry
into download mode » (ESP32-S3 TRM v1.8, p. 1244). Le PHY USB interne est
**partagé** entre l'USB-OTG et l'USB Serial/JTAG, un seul à la fois
(p. 1244-1245) : tant que le firmware tient le PHY en OTG pour le HID, cette
console disparaît.

Vérifié au `kicad-cli` : 0 problème de parité schéma/PCB, 0 élément non
connecté. Les points ouverts restants sont listés dans le
[README](../README.md).

## Côté hôte NixOS

Les règles udev de `mae` épinglent `303a:4001`
(`modules/hardware/kase-dongle.nix`) sur trois sujets : accès hidraw non-root,
accès CCID pour `scdaemon`, et symlink `/dev/ttyKASE_DONGLE`. Le firmware doit
donc présenter un device composite **HID + CDC-ACM + CCID** sous ce couple. S'il
change un jour, ce fichier doit changer dans le même mouvement — il en existe
sept copies identiques dans les variantes de la config.

## Sources

- congatec AN43, *M.2 Pinout Descriptions and Reference Designs*, p. 9 —
  `https://www.congatec.com/fileadmin/user_upload/Documents/Application_Notes/AN43_M.2_Pinout_Descriptions_and_Reference_Designs.pdf`
- Espressif, *ESP32-S3 Technical Reference Manual* v1.8, p. 1244-1245
- STMicroelectronics, *LD1117 datasheet* —
  `https://www.st.com/resource/en/datasheet/ld1117.pdf`
- `timonsku/RP2040-M.2` — `https://github.com/timonsku/RP2040-M.2`
- Hackaday, *M.2 For Hackers – Expand Your Laptop* —
  `https://hackaday.com/2022/10/27/m-2-for-hackers-expand-your-laptop/`
- `danielewood/sierra-wireless-modems` —
  `https://github.com/danielewood/sierra-wireless-modems`
- Fibocom, *L860-GL Hardware User Manual*, p. 29 (PCIe Gen2, USB2 en debug)
- ThinkWiki, *Problem with unauthorized MiniPCI network card* —
  `https://www.thinkwiki.org/wiki/Problem_with_unauthorized_MiniPCI_network_card`
- Lenovo, *ThinkPad P16 Gen 1 Hardware Maintenance Manual* —
  `https://download.lenovo.com/pccbbs/mobiles_pdf/p16_gen1_hmm_en.pdf`
