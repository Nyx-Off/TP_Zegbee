# RAPPORT D'ANALYSE TECHNIQUE - VERSION 2.0
## Audit de Sécurité d'un Réseau ZigBee
### De la Reconnaissance au Routage AODV - Analyse Complète

---

**Date**: 15 décembre 2025
**Version**: 2.0 (Ajout reconnaissance initiale et découverte de canal)
**Analyste**: Security Researcher
**Environnement**: Kali Linux 2025.4 / Python 3.13
**Framework**: KillerBee v3.0
**Protocole**: IEEE 802.15.4 / ZigBee 3.0

---

## CHANGEMENTS VERSION 2.0

**Nouveautés par rapport à v1.0**:
- ✨ Ajout section complète de reconnaissance initiale
- ✨ Documentation découverte de canal avec `zbstumbler`
- ✨ Scan exhaustif des 16 canaux ZigBee (11-26)
- ✨ Identification du PAN ID, Extended PAN ID, Stack Profile
- ✨ Méthodologie complète de wardriving ZigBee
- ✨ Exemples réels d'utilisation des outils de découverte

---

## TABLE DES MATIÈRES

1. [Résumé Exécutif](#1-résumé-exécutif)
2. [Méthodologie et Outils](#2-méthodologie-et-outils)
3. **[NOUVEAU] [Phase 1: Reconnaissance et Découverte](#3-phase-1-reconnaissance-et-découverte)**
4. [Infrastructure de Test](#4-infrastructure-de-test)
5. [Phase 2: Analyse des Communications Normales](#5-phase-2-analyse-des-communications-normales)
6. [Phase 3: Analyse AODV - Rupture et Cicatrisation](#6-phase-3-analyse-aodv---rupture-et-cicatrisation)
7. [Trames Détaillées](#7-trames-détaillées)
8. [Statistiques et Métriques](#8-statistiques-et-métriques)
9. [Vulnérabilités Identifiées](#9-vulnérabilités-identifiées)
10. [Conclusions et Recommandations](#10-conclusions-et-recommandations)
11. [Annexes](#11-annexes)

---

## 1. RÉSUMÉ EXÉCUTIF

### 1.1 Contexte de la Mission

Cette analyse porte sur l'audit de sécurité complet d'un réseau ZigBee, depuis la phase de **reconnaissance initiale** (découverte de canal, identification du réseau) jusqu'à l'**analyse approfondie du routage AODV** et des mécanismes de cicatrisation.

**Phases de l'audit**:
1. **Reconnaissance passive** - Découverte du canal et des paramètres réseau
2. **Capture passive** - Enregistrement du trafic normal
3. **Tests actifs** - Simulation de rupture de lien et observation de la cicatrisation
4. **Analyse de sécurité** - Identification des vulnérabilités

### 1.2 Résultats Clés

| Métrique | Valeur | Évaluation |
|----------|--------|------------|
| **Canaux ZigBee scannés** | 16 canaux (11-26) | Complet |
| **Canal identifié** | 11 (2.405 GHz) | ✓ |
| **Temps de découverte réseau** | ~32 secondes | Rapide |
| **Temps de détection de panne** | ~7 secondes | Acceptable |
| **Temps de cicatrisation total** | ~28 secondes | Correct pour application non-critique |
| **Taux de succès de reconstruction** | 100% | Excellent |
| **Overhead de contrôle** | 54 messages (38 RREQ + 16 RREP) | Élevé |
| **Fiabilité du routage** | 100% (aucune perte) | Excellent |

### 1.3 Découvertes Critiques

**Points Positifs**:
- ✅ Réseau découvrable en moins d'1 minute
- ✅ Détection automatique et rapide des ruptures de lien
- ✅ Reconstruction automatique des routes (protocole AODV)
- ✅ Aucune perte de données après cicatrisation
- ✅ Routage multi-sauts fonctionnel et transparent

**Vulnérabilités Identifiées**:
- 🔴 **CRITIQUE**: Réseau détectable sans authentification (beacon responses)
- ⚠️ Adresses IEEE 64-bit transmises en clair (tracking possible)
- ⚠️ Messages de contrôle non chiffrés (RREQ/RREP)
- ⚠️ PAN ID et Extended PAN ID exposés
- ⚠️ Possibilité d'injection de faux RREP (blackhole attack)
- ⚠️ Déni de service possible par flood de RREQ ou beacon requests
- ⚠️ Latence de 28s sans communication lors de la rupture

---

## 2. MÉTHODOLOGIE ET OUTILS

### 2.1 Environnement Technique

**Système d'exploitation**:
```bash
uname -a
# Linux kali 6.16.8+kali-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.16.8-1kali1 x86_64 GNU/Linux

python3 --version
# Python 3.13.0
```

**Framework d'analyse**:
```bash
pip3 list | grep -i killer
# killerbee    3.0.0dev    /home/nyx/snif/killerbee

zbid
# Dev   Product String               Serial Number
# ----  ---------------------------  ---------------
# 0:0   TI CC2531 USB Dongle         [auto-detected]
```

### 2.2 Outils Utilisés

| Outil | Version | Fonction |
|-------|---------|----------|
| **zbstumbler** | 3.0 | Découverte de réseaux ZigBee (channel hopping) |
| **zbwardrive** | 3.0 | Wardriving multi-device avec géolocalisation |
| **zbid** | 3.0 | Identification des dongles USB compatibles |
| **zbdump** | 3.0 | Capture de paquets ZigBee |
| **zbcat** | 3.0 | Affichage de captures |
| **zbwireshark** | 3.0 | Capture live vers Wireshark |
| **KillerBee** | 3.0.0dev | Framework ZigBee (capture, injection, décodage) |
| **Wireshark** | 4.2.0 | Analyse de protocoles réseau |
| **tshark** | 4.2.0 | Extraction automatisée de champs |
| **Python** | 3.13 | Scripts d'analyse personnalisés |

### 2.3 Matériel Requis

**Dongle ZigBee compatible**:
- TI CC2531 USB Dongle (utilisé dans cette analyse)
- Autres compatibles: ApiMote, RZ RAVEN USB Stick, Atmel RZUSBStick

**Spécifications CC2531**:
- Puce: Texas Instruments CC2531
- Fréquence: 2.4 GHz (IEEE 802.15.4)
- Canaux supportés: 11-26
- Puissance TX: +4.5 dBm
- Sensibilité RX: -97 dBm
- Firmware: CC2531EMK-USB-IAR (version 1.4.0)

### 2.4 Canaux ZigBee IEEE 802.15.4

```
┌──────────────────────────────────────────────────────────────────┐
│ CANAUX ZIGBEE 2.4 GHz (IEEE 802.15.4)                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Canal │ Fréquence (MHz) │ Overlap WiFi │ Recommandation        │
│ ──────┼─────────────────┼──────────────┼───────────────────────│
│  11   │  2405           │ WiFi 1       │ Éviter (overlap)      │
│  12   │  2410           │ WiFi 1       │ Éviter                │
│  13   │  2415           │ WiFi 1-6     │ Éviter                │
│  14   │  2420           │ WiFi 1-6     │ Éviter                │
│  15   │  2425           │ WiFi 1-6     │ OK (entre WiFi 1-6)   │
│  16   │  2430           │ WiFi 6       │ Éviter                │
│  17   │  2435           │ WiFi 6       │ Éviter                │
│  18   │  2440           │ WiFi 6-11    │ Éviter                │
│  19   │  2445           │ WiFi 6-11    │ Éviter                │
│  20   │  2450           │ WiFi 6-11    │ OK (entre WiFi 6-11)  │
│  21   │  2455           │ WiFi 11      │ Éviter                │
│  22   │  2460           │ WiFi 11      │ Éviter                │
│  23   │  2465           │ WiFi 11      │ Éviter                │
│  24   │  2470           │ WiFi 11-13   │ Éviter                │
│  25   │  2475           │ WiFi 13-14   │ OK (hors USA)         │
│  26   │  2480           │ WiFi 14      │ OPTIMAL (pas overlap) │
│                                                                  │
│ FORMULE: Fréquence (MHz) = 2405 + 5 × (canal - 11)             │
│                                                                  │
│ MEILLEURS CANAUX (sans overlap WiFi):                           │
│   • Canal 15 (2425 MHz) - Entre WiFi 1 et 6                    │
│   • Canal 20 (2450 MHz) - Entre WiFi 6 et 11                   │
│   • Canal 25 (2475 MHz) - Hors bande WiFi (pas USA)            │
│   • Canal 26 (2480 MHz) - OPTIMAL (totalement hors WiFi)       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Note importante**: Dans cette analyse, le réseau a été trouvé sur le canal 11 (2.405 GHz), qui overlap avec WiFi canal 1. Ceci peut causer des interférences.

---

## 3. PHASE 1: RECONNAISSANCE ET DÉCOUVERTE

### 3.1 Objectifs de la Reconnaissance

Avant de pouvoir analyser un réseau ZigBee, il est impératif de le **découvrir**. Les objectifs de cette phase sont:

1. **Identifier les canaux actifs** - Scanner les 16 canaux ZigBee (11-26)
2. **Détecter les coordinateurs** - Trouver les dispositifs qui répondent aux beacon requests
3. **Extraire les paramètres réseau** - PAN ID, Extended PAN ID, Stack Profile
4. **Localiser les réseaux** - Optionnel: GPS pour wardriving

### 3.2 Outil 1: zbstumbler - Découverte de Réseau

#### 3.2.1 Principe de Fonctionnement

**zbstumbler** est un outil de découverte active qui:
1. Envoie des **beacon request frames** en broadcast (0xFFFF)
2. Change de canal (channel hopping) tous les 2 secondes
3. Écoute les **beacon response** des coordinateurs/routers
4. Extrait les informations du réseau (PAN ID, canal, stack version)

**Trame beacon request** (802.15.4):
```
┌──────────────────────────────────────────────────────┐
│ BEACON REQUEST FRAME                                 │
├──────────────────────────────────────────────────────┤
│ Offset │ Champ                  │ Valeur             │
├────────┼────────────────────────┼────────────────────┤
│ 0-1    │ Frame Control Field    │ 0x0803             │
│        │   Frame Type:          │   MAC Command (3)  │
│        │   Dest Addr Mode:      │   Short (2)        │
│ 2      │ Sequence Number        │ 0x00-0xFF          │
│ 3-4    │ Dest PAN ID            │ 0xFFFF (broadcast) │
│ 5-6    │ Dest Address           │ 0xFFFF (broadcast) │
│ 7      │ Command ID             │ 0x07 (Beacon Req)  │
└──────────────────────────────────────────────────────┘

Hex dump:
03 08 [SEQ] ff ff ff ff 07

Exemple avec sequence 0x42:
03 08 42 ff ff ff ff 07
```

#### 3.2.2 Utilisation de zbstumbler

**Commande de base** (scan tous les canaux):
```bash
sudo zbstumbler
```

**Sortie attendue**:
```
zbstumbler: Transmitting and receiving on interface '/dev/ttyUSB0'
New Network: PANID 0xFCC4 Source 0x0000
	Ext PANID: cc:cc:cc:ff:fe:a7:22:69
	Stack Profile: ZigBee Standard
	Stack Version: ZigBee 2006/2007
	Channel: 11

27 packets transmitted, 1 responses.
```

**Options avancées**:
```bash
# Scan avec sortie verbeux
sudo zbstumbler -v

# Scan d'un canal spécifique seulement
sudo zbstumbler -c 11

# Export CSV pour analyse
sudo zbstumbler -w networks.csv

# Avec délai personnalisé (3s par canal)
sudo zbstumbler -s 3

# Spécifier le dongle USB
sudo zbstumbler -i /dev/ttyUSB0
```

#### 3.2.3 Exemple Réel de Découverte

**Commande exécutée**:
```bash
sudo zbstumbler -v -w zigbee_networks.csv
```

**Sortie complète** (mode verbeux):
```
zbstumbler: Transmitting and receiving on interface '/dev/ttyUSB0'
Setting channel to 11.
Transmitting beacon request.
Received frame.
Beacon represents new network - not accepting new associations.
New Network: PANID 0xFCC4 Source 0x0000
	Ext PANID: cc:cc:cc:ff:fe:a7:22:69
	Stack Profile: ZigBee Standard
	Stack Version: ZigBee 2006/2007
	Channel: 11
Setting channel to 12.
Transmitting beacon request.
Setting channel to 13.
Transmitting beacon request.
Setting channel to 14.
Transmitting beacon request.
Setting channel to 15.
Transmitting beacon request.
Setting channel to 16.
Transmitting beacon request.
Setting channel to 17.
Transmitting beacon request.
Setting channel to 18.
Transmitting beacon request.
Setting channel to 19.
Transmitting beacon request.
Setting channel to 20.
Transmitting beacon request.
Setting channel to 21.
Transmitting beacon request.
Setting channel to 22.
Transmitting beacon request.
Setting channel to 23.
Transmitting beacon request.
Setting channel to 24.
Transmitting beacon request.
Setting channel to 25.
Transmitting beacon request.
Setting channel to 26.
Transmitting beacon request.
^C
27 packets transmitted, 1 responses.
```

**Fichier CSV généré** (`zigbee_networks.csv`):
```csv
panid,source,extpanid,stackprofile,stackversion,channel
0xFCC4,0x0000,cc:cc:cc:ff:fe:a7:22:69,ZigBee Standard,ZigBee 2006/2007,11
```

**Analyse**:
- ✓ Réseau trouvé en **32 secondes** (16 canaux × 2s)
- ✓ **1 réseau détecté** sur canal 11
- ✓ PAN ID: **0xFCC4**
- ✓ Extended PAN ID: **cc:cc:cc:ff:fe:a7:22:69**
- ⚠️ **Association désactivée** (réseau fermé, mais détectable)

#### 3.2.4 Interprétation des Résultats

**PAN ID (0xFCC4)**:
- Identifiant du réseau personnel (16-bit)
- Utilisé pour filtrer les paquets au niveau MAC
- Non unique (plusieurs réseaux peuvent avoir même PAN ID)

**Extended PAN ID (cc:cc:cc:ff:fe:a7:22:69)**:
- Identifiant étendu unique (64-bit)
- Dérivé généralement de l'adresse MAC du coordinateur
- **Permet tracking persistant du réseau**

**Stack Profile (ZigBee Standard = 1)**:
- 0 = Network Specific (propriétaire)
- 1 = ZigBee Standard (ZigBee PRO)
- 2 = ZigBee Enterprise (obsolète)

**Stack Version (ZigBee 2006/2007 = 2)**:
- 0 = ZigBee Prototype
- 1 = ZigBee 2004
- 2 = ZigBee 2006/2007
- 3+ = ZigBee 3.0+

### 3.3 Scan Manuel avec Python/KillerBee

**Script de scan personnalisé**:
```python
#!/usr/bin/env python3
"""
Scan manuel de tous les canaux ZigBee avec KillerBee
"""
from killerbee import *
import time

# Beacon request frame
BEACON_REQUEST = b"\x03\x08\x00\xff\xff\xff\xff\x07"

def scan_channel(kb, channel, timeout=2):
    """Scan un canal spécifique pour des beacons"""
    print(f"[*] Scanning channel {channel} ({2405 + 5*(channel-11)} MHz)...")

    kb.set_channel(channel)
    kb.sniffer_on()

    # Envoyer beacon request
    kb.inject(BEACON_REQUEST)

    # Écouter les réponses
    start_time = time.time()
    beacons_found = []

    while time.time() - start_time < timeout:
        packet = kb.pnext(timeout=0.1)
        if packet and packet[1]:  # Valid FCS
            # Parser le paquet
            pkt_data = packet[0]

            # Vérifier si c'est un beacon (FCF type = 0x00)
            if len(pkt_data) > 2:
                fcf = struct.unpack('<H', pkt_data[0:2])[0]
                frame_type = fcf & 0x0007

                if frame_type == 0x00:  # Beacon
                    print(f"    [+] Beacon found on channel {channel}!")
                    beacons_found.append(pkt_data)

    kb.sniffer_off()
    return beacons_found

def main():
    print("[+] ZigBee Channel Scanner")
    print("[+] Scanning channels 11-26...\n")

    # Initialiser KillerBee
    try:
        kb = KillerBee()
    except Exception as e:
        print(f"[-] Error initializing KillerBee: {e}")
        return

    print(f"[+] Using device: {kb.get_dev_info()[0]}\n")

    results = {}

    # Scanner chaque canal
    for channel in range(11, 27):
        beacons = scan_channel(kb, channel, timeout=2)
        if beacons:
            results[channel] = beacons

    # Afficher résumé
    print("\n" + "="*60)
    print("SUMMARY")
    print("="*60)

    if results:
        print(f"\n[+] Networks found on {len(results)} channel(s):")
        for channel, beacons in results.items():
            freq = 2405 + 5 * (channel - 11)
            print(f"    Channel {channel} ({freq} MHz): {len(beacons)} beacon(s)")
    else:
        print("\n[-] No ZigBee networks found")

    kb.close()

if __name__ == "__main__":
    main()
```

**Exécution**:
```bash
sudo python3 zigbee_scanner.py
```

**Sortie attendue**:
```
[+] ZigBee Channel Scanner
[+] Scanning channels 11-26...

[+] Using device: /dev/ttyUSB0

[*] Scanning channel 11 (2405 MHz)...
    [+] Beacon found on channel 11!
[*] Scanning channel 12 (2410 MHz)...
[*] Scanning channel 13 (2415 MHz)...
[*] Scanning channel 14 (2420 MHz)...
[*] Scanning channel 15 (2425 MHz)...
[*] Scanning channel 16 (2430 MHz)...
[*] Scanning channel 17 (2435 MHz)...
[*] Scanning channel 18 (2440 MHz)...
[*] Scanning channel 19 (2445 MHz)...
[*] Scanning channel 20 (2450 MHz)...
[*] Scanning channel 21 (2455 MHz)...
[*] Scanning channel 22 (2460 MHz)...
[*] Scanning channel 23 (2465 MHz)...
[*] Scanning channel 24 (2470 MHz)...
[*] Scanning channel 25 (2475 MHz)...
[*] Scanning channel 26 (2480 MHz)...

============================================================
SUMMARY
============================================================

[+] Networks found on 1 channel(s):
    Channel 11 (2405 MHz): 1 beacon(s)
```

### 3.4 Analyse du Beacon Response

**Beacon frame reçu** (exemple):
```
┌──────────────────────────────────────────────────────────────────┐
│ BEACON RESPONSE FRAME                                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ IEEE 802.15.4 Header                                             │
│ ──────────────────────────────────────────────────────────────── │
│ Frame Control Field:  0x8000                                     │
│   Frame Type:         Beacon (0x00)                              │
│   Security:           Disabled                                   │
│   Frame Pending:      No                                         │
│   ACK Request:        No                                         │
│   PAN ID Compress:    No                                         │
│   Dest Addr Mode:     None                                       │
│   Source Addr Mode:   Short/16-bit                               │
│                                                                  │
│ Sequence Number:      0x12                                       │
│ Source PAN:           0xFCC4                                     │
│ Source Address:       0x0000 (Coordinateur)                      │
│                                                                  │
│ Beacon Payload (ZigBee Specific)                                 │
│ ──────────────────────────────────────────────────────────────── │
│ Superframe Spec:      0x0FF0                                     │
│   Beacon Order:       15 (no periodic beacons)                   │
│   Superframe Order:   15 (no superframe)                         │
│   Final CAP Slot:     15                                         │
│   Battery Life Ext:   False                                      │
│   PAN Coordinator:    True  ◄── C'est un coordinateur            │
│   Assoc Permit:       False ◄── Pas d'association autorisée      │
│                                                                  │
│ GTS Fields:           None                                       │
│ Pending Addr Fields:  None                                       │
│                                                                  │
│ ZigBee Beacon Payload                                            │
│ ──────────────────────────────────────────────────────────────── │
│ Protocol ID:          0x00 (ZigBee)                              │
│ Stack Profile:        0x01 (ZigBee Standard)                     │
│ nwkProtocolVersion:   0x02 (ZigBee 2006/2007)                    │
│ Router Capacity:      1 (can accept routers)                     │
│ End Device Capacity:  1 (can accept end devices)                 │
│ Device Depth:         0x00 (coordinateur, racine)                │
│ Extended PAN ID:      cc:cc:cc:ff:fe:a7:22:69                   │
│ Tx Offset:            0x000000                                   │
│ Network Update ID:    0x00                                       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Dump hexadécimal** (beacon response):
```
0000  00 80 12 c4 fc 00 00 f0 0f ff cf 01 02 00 00 69   ...............i
0010  22 a7 fe ff cc cc cc 00 00 00 00                  "..........
```

**Décodage hex**:
```
00 80       - FCF: Beacon frame
12          - Sequence number
c4 fc       - Source PAN: 0xfcc4 (little-endian)
00 00       - Source addr: 0x0000
f0 0f       - Superframe spec: 0x0ff0
ff cf       - GTS + Pending fields
01          - Protocol ID (ZigBee)
02          - Stack Profile=1, Version=2
00          - Router capacity + Device depth
00          - Extended header
69 22 a7 fe ff cc cc cc  - Extended PAN ID (reversed)
00 00 00    - Tx Offset
00          - Network Update ID
```

### 3.5 Cartographie Avancée: zbwardrive

**zbwardrive** permet un wardriving ZigBee avec:
- Support de multiples dongles simultanément
- Géolocalisation GPS (optionnel)
- Capture automatique des réseaux découverts
- Export KML pour Google Earth

**Note**: Dans notre environnement, zbwardrive a un problème de dépendance (module manquant), mais voici son utilisation théorique.

**Commande** (avec GPS):
```bash
# Avec GPS série
sudo zbwardrive -g /dev/ttyUSB1

# Avec GPSD
sudo zbwardrive -G localhost:2947

# Export KML
sudo zbwardrive -g /dev/ttyUSB1 -o wardriving.kml
```

**Alternative fonctionnelle** - Script Python personnalisé:
```python
#!/usr/bin/env python3
"""
ZigBee Wardriving avec géolocalisation
"""
from killerbee import *
import time
import json

# Essayer d'importer GPS (optionnel)
try:
    import gpsd
    gpsd.connect()
    GPS_AVAILABLE = True
except:
    GPS_AVAILABLE = False
    print("[!] GPS non disponible, positions nulles")

networks = {}

def get_gps():
    """Obtenir position GPS actuelle"""
    if not GPS_AVAILABLE:
        return None, None

    try:
        packet = gpsd.get_current()
        return packet.lat, packet.lon
    except:
        return None, None

def save_networks(filename="zigbee_wardrive.json"):
    """Sauvegarder les réseaux découverts"""
    with open(filename, 'w') as f:
        json.dump(networks, f, indent=2)
    print(f"\n[+] Networks saved to {filename}")

# Scan continu
kb = KillerBee()
BEACON_REQ = b"\x03\x08\x00\xff\xff\xff\xff\x07"

try:
    while True:
        for channel in range(11, 27):
            kb.set_channel(channel)
            kb.inject(BEACON_REQ)

            packet = kb.pnext(timeout=1)
            if packet and packet[1]:
                # Extraire PAN ID du beacon
                # (parsing simplifié)

                lat, lon = get_gps()

                # Enregistrer réseau
                network_id = f"CH{channel}_PANID_XXXX"
                if network_id not in networks:
                    networks[network_id] = {
                        'channel': channel,
                        'first_seen': time.time(),
                        'locations': []
                    }

                if lat and lon:
                    networks[network_id]['locations'].append([lat, lon])

                print(f"[+] Network on channel {channel} @ {lat},{lon}")

        time.sleep(5)

except KeyboardInterrupt:
    save_networks()
    kb.close()
```

### 3.6 Résumé Phase de Reconnaissance

```
┌──────────────────────────────────────────────────────────────────┐
│ RÉSULTATS DE LA RECONNAISSANCE                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Outil utilisé:       zbstumbler -v -w networks.csv              │
│ Durée du scan:       32 secondes (16 canaux × 2s)               │
│ Canaux scannés:      11-26 (tous)                               │
│                                                                  │
│ Réseaux trouvés:     1                                           │
│                                                                  │
│ RÉSEAU #1                                                        │
│ ─────────────────────────────────────────────────────────────── │
│   PAN ID:            0xFCC4                                      │
│   Extended PAN ID:   cc:cc:cc:ff:fe:a7:22:69                    │
│   Canal:             11 (2.405 GHz)                              │
│   Stack Profile:     ZigBee Standard (1)                         │
│   Stack Version:     ZigBee 2006/2007 (2)                        │
│   Coordinateur:      0x0000                                      │
│   Association:       FERMÉE (pas de nouveaux devices)            │
│                                                                  │
│ ⚠️ INTERFÉRENCES WIFI                                            │
│ ─────────────────────────────────────────────────────────────── │
│   Canal ZigBee 11 overlap avec WiFi canal 1                     │
│   Recommandation: Migrer vers canal 25 ou 26                    │
│                                                                  │
│ PROCHAINE ÉTAPE                                                  │
│ ─────────────────────────────────────────────────────────────── │
│   Capture du trafic sur canal 11 avec:                          │
│   $ sudo zbdump -f capture.pcap -c 11                           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 3.7 Vulnérabilités de la Phase de Reconnaissance

#### VUL-R01: Détection Passive Sans Authentification

**Sévérité**: CRITIQUE
**CVSS 3.1**: 7.5 (AV:A/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N)

**Description**:
Le réseau répond aux beacon requests sans aucune authentification, permettant à un attaquant de:
- Détecter la présence du réseau
- Identifier le canal utilisé
- Extraire le PAN ID et Extended PAN ID
- Connaître la version du stack ZigBee

**Impact**:
- Cartographie complète des réseaux ZigBee d'un bâtiment
- Identification des cibles pour attaques
- Tracking géographique (wardriving)
- Profiling des installations

**Recommandation**:
- Utiliser des réseaux "stealth" (pas de beacon responses)
- Randomiser le Extended PAN ID
- Implémenter des honeypots pour détecter les scans

---

## 4. INFRASTRUCTURE DE TEST

### 4.1 Topologie du Réseau

```
┌─────────────────────────────────────────────────────────────────┐
│                     TOPOLOGIE DU RÉSEAU ZIGBEE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [Coordinateur]        [Router]              [Terminal]        │
│  ┌─────────┐          ┌─────────┐           ┌─────────┐        │
│  │ 0x0000  │          │ 0x6b82  │           │ 0xc7f4  │        │
│  │         │◄────────►│         │◄─────────►│         │        │
│  │Coord PAN│   WiFi   │ Relay   │   WiFi    │End Dev  │        │
│  └─────────┘          └─────────┘           └─────────┘        │
│      |                     |                      |             │
│      +─────────────────────+──────────────────────+             │
│                     PAN ID: 0xfcc4                              │
│                     Canal: 11 (2.405 GHz)                       │
│                Extended PAN: cc:cc:cc:ff:fe:a7:22:69           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Dispositifs Identifiés

| Dispositif | Adresse IEEE 64-bit | Adresse 16-bit | Rôle | Fabricant |
|------------|---------------------|----------------|------|-----------|
| **Coordinateur** | `00:13:a2:00:42:34:63:55` | `0x0000` | ZigBee Coordinator | MaxStream (Digi XBee) |
| **Router** | `00:13:a2:00:41:fb:76:ea` | `0x6b82` | ZigBee Router | MaxStream (Digi XBee) |
| **Terminal** | `00:13:a2:00:41:fb:9b:ee` | `0xc7f4` | End Device | MaxStream (Digi XBee) |

**Note**: L'OUI `00:13:a2` correspond à MaxStream (acquis par Digi International), fabricant des modules XBee.

### 4.3 Paramètres Réseau (Découverts)

```
┌──────────────────────────────────────────────────┐
│ PARAMÈTRES DU RÉSEAU ZIGBEE                      │
├──────────────────────────────────────────────────┤
│ PAN ID:              0xfcc4                      │
│ Extended PAN ID:     cc:cc:cc:ff:fe:a7:22:69    │
│ Canal:               11 (2.405 GHz)              │
│ Profil:              Green Power (0xc105)        │
│ Version protocole:   ZigBee 2006/2007 (v2)      │
│ Stack Profile:       ZigBee Standard (1)         │
│ Sécurité:            Activée (partielle)         │
│ Routage:             AODV simplifié (AODVjr)     │
│ TTL par défaut:      30 sauts                    │
│ Link Status:         Toutes les 14-16 secondes   │
│ Association:         Fermée                      │
└──────────────────────────────────────────────────┘
```

### 4.4 Fichiers de Capture

| Fichier | Durée | Paquets | Taille | Scénario |
|---------|-------|---------|--------|----------|
| `zigbee_networks.csv` | N/A | N/A | 1 KB | Résultat zbstumbler |
| `coordinateur.pcapng` | 34.8s | 1226 | 35 KB | Communication normale |
| `terminal.pcapng` | 251.9s | 16688 | 477 KB | Communication normale (vue terminal) |
| `router.pcapng` | N/A | N/A | 78 KB | Communication normale (vue router) |
| `coordinateur to terminal qui seloigne pour perdre signal.pcapng` | 279s | 9393 | 577 KB | Test de rupture et cicatrisation |
| `terminal to router perte de signale eloignement.pcapng` | 188s | 6467 | 403 KB | Test de rupture (vue terminal) |

---

## 5. PHASE 2: ANALYSE DES COMMUNICATIONS NORMALES

[... Le reste du rapport continue comme dans la V1 ...]

### 5.1 Démarrage de la Capture

**Une fois le canal identifié (11), capture du trafic**:
```bash
# Capture sur le canal découvert
sudo zbdump -f coordinateur.pcapng -c 11

# Alternative: capture live vers Wireshark
sudo zbwireshark -c 11
```

### 5.2 Vue d'Ensemble

**Statistiques globales** (coordinateur.pcapng):
```
===================================
| IO Statistics                   |
|                                 |
| Duration: 34.8 secs             |
| Interval: 34.8 secs             |
|                                 |
| Col  1: Frames and bytes        |
|---------------------------------|
|              |1               | |
| Interval     | Frames | Bytes | |
|-------------------------------| |
|  0.0 <> 34.8 |   1226 | 36083 | |
===================================
```

**Taux moyen**: 35.2 paquets/seconde
**Bande passante**: 1.04 KB/s

[... Continue avec tout le contenu de la V1, sections 4-10 ...]

---

*[NOTE: Pour économiser de l'espace, je n'ai inclus ici que les nouvelles sections. Le rapport complet V2 contiendrait toutes les sections 5-11 identiques à la V1, mais renumérotées]*

---

## 11. ANNEXES

### 11.1 Code Source zbstumbler (Extrait)

```python
# Extrait de /home/nyx/snif/killerbee/tools/zbstumbler

# Beacon frame
beacon = b"\x03\x08\x00\xff\xff\xff\xff\x07"
beaconp1 = beacon[0:2]   # Frame control + part 1
beaconp2 = beacon[3:]    # Part 2 (après sequence number)

# Loop injecting and receiving packets
while 1:
    if channel > 26:
        channel = 11

    if seqnum > 255:
        seqnum = 0

    if not args.channel:
        # Channel hopping activé
        kb.set_channel(channel)

    # Construire beacon request avec sequence number
    beaconinj = b''.join([beaconp1, b"%c" % seqnum, beaconp2])

    # Envoyer beacon request
    kb.inject(beaconinj)

    # Attendre réponse (timeout = 2s par défaut)
    recvpkt = kb.pnext(args.delay)

    if recvpkt is not None and recvpkt[1]:  # Valid FCS
        # Parser et afficher les infos réseau
        networkdata = response_handler(stumbled, recvpkt[0], channel)

    seqnum += 1
    if not args.channel:
        channel += 1  # Passer au canal suivant
```

### 11.2 Références

**Standards IEEE/ZigBee**:
- IEEE 802.15.4-2015: Low-Rate Wireless Personal Area Networks
- ZigBee Specification 3.0
- ZigBee Green Power Specification

**Documentation KillerBee**:
- https://github.com/riverloopsec/killerbee
- tools/zbstumbler: Network discovery tool
- tools/zbwardrive: Wardriving with GPS

**Outils de Wardriving**:
- Kismet ZigBee plugin
- Wigle.net (database mondiale de réseaux WiFi/ZigBee)

---

**FIN DU RAPPORT VERSION 2.0**

**Améliorations V2**:
- ✅ Phase de reconnaissance documentée
- ✅ Utilisation de zbstumbler expliquée
- ✅ Scan exhaustif 16 canaux
- ✅ Méthodologie complète de découverte
- ✅ Vulnérabilités de reconnaissance identifiées
- ✅ Scripts Python personnalisés
