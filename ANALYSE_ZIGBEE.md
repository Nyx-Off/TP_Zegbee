# Analyse des Communications ZigBee - Coordinateur et Terminal

## Vue d'ensemble

Cette analyse examine les communications ZigBee capturées entre un coordinateur et des terminaux (end devices) dans un réseau ZigBee. Deux fichiers de capture ont été analysés:

- **coordinateur.pcapng**: 2168 paquets sur 44.7 secondes (60.7 KB)
- **terminal.pcapng**: 16688 paquets sur 251.9 secondes (477 KB)

## Dispositifs Identifiés

### Coordinateur (Network Coordinator)
- **Adresse IEEE 802.15.4 (64-bit)**: `00:13:a2:00:42:34:63:55`
- **Adresse courte (16-bit)**: `0x0000` (adresse réservée au coordinateur)
- **Rôle**: Coordinateur du réseau ZigBee, gère le routage et la topologie

### Router (ZigBee Router)
- **Adresse IEEE 802.15.4 (64-bit)**: `00:13:a2:00:41:fb:76:ea`
- **Adresse courte (16-bit)**: `0x6b82`
- **Rôle**: Router intermédiaire - relaie les paquets entre le coordinateur et le terminal

### Terminal / End Device
- **Adresse IEEE 802.15.4 (64-bit)**: `00:13:a2:00:41:fb:9b:ee`
- **Adresse courte (16-bit)**: `0xc7f4`
- **Rôle**: End device, communique avec le coordinateur via le router

### Dispositif #3
- **Adresse IEEE 802.15.4 (64-bit)**: `00:13:a2:00:42:53:37:ab`
- **Adresse courte (16-bit)**: Non déterminé
- **Rôle**: Possible routeur ou coordinateur secondaire

---

## Méthodes d'Analyse Utilisées

### 1. Extraction des Adresses Étendues (Extended Addresses)

Les adresses 64-bit sont extraites de la couche **ZigBee NWK** lorsque le flag `ZBEE_NWK_FCF_EXT_SOURCE` ou `ZBEE_NWK_FCF_EXT_DEST` est activé dans le Frame Control Field.

**Dans KillerBee** (zigbeedecode.py:80-85):
```python
# Check if the SA bit is set in the frame control field
if (fc & ZBEE_NWK_FCF_EXT_SOURCE) != 0:
    pktchop.append(packet[offset:offset+8])  # Extraction des 8 octets de l'adresse source
    offset+=8
```

**Filtre Wireshark utilisé**:
```
zbee_nwk.src64 == 00:13:a2:00:42:34:63:55  # Pour filtrer le coordinateur
zbee_nwk.src64 == 00:13:a2:00:41:fb:9b:ee  # Pour filtrer le terminal
```

### 2. Mapping Adresses Courtes ↔ Adresses Étendues

Les adresses courtes (16-bit) sont attribuées dynamiquement par le coordinateur lors de l'association:

| Adresse IEEE 64-bit | Adresse Courte | Type |
|---------------------|----------------|------|
| 00:13:a2:00:42:34:63:55 | 0x0000 | Coordinateur |
| 00:13:a2:00:41:fb:9b:ee | 0xc7f4 | End Device |
| 00:13:a2:00:41:fb:76:ea | 0x6b82 | End Device |

### 3. Analyse des Protocoles et Clusters

#### Profil ZigBee
- **Profile ID**: `0xc105` (Green Power Profile)
  - Utilisé pour les dispositifs à très faible consommation
  - Communication unidirectionnelle principalement

#### Clusters APS Identifiés

| Cluster ID | Nom | Description | Fréquence (Coord) | Fréquence (Term) |
|------------|-----|-------------|-------------------|------------------|
| 0x0021 | Green Power | Commandes Green Power | 338 paquets | 3506 paquets |
| 0x00a1 | RSSI Location | Localisation par RSSI | 349 paquets | 2654 paquets |
| 0x0000 | ZDO | Device/Service Discovery | ~6 paquets | ~34 paquets |

---

## Analyse des Communications

### Distribution des Types de Paquets

#### Coordinateur (coordinateur.pcapng)
```
Type de Paquet    Nombre    Pourcentage
ACK (0x0002)      1063      49.0%
Data (0x0001)     402       18.5%
Cluster 0x00a1    349       16.1%
Cluster 0x0021    338       15.6%
Command           15        0.7%
```

#### Terminal (terminal.pcapng)
```
Type de Paquet    Nombre    Pourcentage
ACK (0x0002)      7969      47.7%
Data (0x0001)     2382      14.3%
Cluster 0x0021    3506      21.0%
Cluster 0x00a1    2654      15.9%
Command           177       1.1%
```

### Commandes NWK Observées

#### Commande 0x01 - Network Status
- **Fonction**: Annonce de présence du dispositif sur le réseau
- **Observations**:
  - Envoyée par tous les dispositifs lors du démarrage
  - Utilisée pour maintenir la table de voisinage

**Exemples identifiés**:
- Paquet #1 (terminal.pcapng): Device 00:13:a2:00:42:53:37:ab annonce sa présence
- Paquet #399-435: Multiple announcements du device 0x42:53:37:ab

#### Commande 0x02 - Leave
- **Fonction**: Demande de départ du réseau
- **Observations**:
  - Device → Coordinateur: Annonce de départ
  - Coordinateur → Device: Confirmation de départ

**Flux observé**:
```
Paquet #395: Coordinateur (00:13:a2:00:42:34:63:55)
             → Device (00:13:a2:00:42:53:37:ab)
             Type: Leave Request (0x02)
```

#### Commande 0x08 - Link Status
- **Fonction**: Mise à jour de la qualité des liens entre voisins
- **Observations**:
  - Envoyée périodiquement par tous les dispositifs
  - Contient les coûts de routage et la qualité du lien

**Exemples identifiés**:
```
Paquet #304: Coordinateur → Broadcast (Link Status)
Paquet #337: Terminal 0x41:fb:9b:ee → Broadcast
Paquet #382: Device 0x41:fb:76:ea → Broadcast
```

---

## Flux de Communication Coordinateur ↔ Terminal

### Pattern de Communication Typique

```
[Coordinateur 0x0000] ──────────────────────────> [Terminal 0xc7f4]
   Cluster 0x0021 (Green Power Command)

[Terminal 0xc7f4] ──────────────────────────────> [Coordinateur 0x0000]
   ACK (0x0002)

[Terminal 0xc7f4] ──────────────────────────────> [Coordinateur 0x0000]
   Cluster 0x00a1 (RSSI Location Data)

[Coordinateur 0x0000] ──────────────────────────> [Terminal 0xc7f4]
   ACK (0x0002)
```

### Séquence d'Échange Observée (coordinateur.pcapng)

| Paquet | Source | Destination | Protocole | Description |
|--------|--------|-------------|-----------|-------------|
| #1 | Coord (42:34:63:55) | Term (41:fb:9b:ee) | ZCL/Green Power | Commande Green Power |
| #3 | Coord (42:34:63:55) | Term (41:fb:9b:ee) | ZCL/Green Power | Commande Green Power |
| #5 | Term (41:fb:9b:ee) | Coord (42:34:63:55) | APS | ACK |
| #7 | Term (41:fb:9b:ee) | Coord (42:34:63:55) | APS | ACK |
| #9 | Term (41:fb:9b:ee) | Coord (42:34:63:55) | ZCL | Réponse avec données |
| #11 | Term (41:fb:9b:ee) | Coord (42:34:63:55) | ZCL | Données RSSI |
| #13 | Coord (42:34:63:55) | Term (41:fb:9b:ee) | APS | ACK |
| #15 | Coord (42:34:63:55) | Term (41:fb:9b:ee) | APS | ACK |

### Caractéristiques de la Communication

1. **Mode de Transmission**: Unicast avec ACK requis
2. **Fréquence**: Communications régulières toutes les ~1-2 secondes
3. **Taille des paquets**: Moyenne de 28 octets (hors ACK)
4. **Ratio ACK/Data**: ~50% des paquets sont des ACK
5. **Fiabilité**: Présence systématique d'ACK pour chaque trame de données

---

## Analyse de la Couche Réseau ZigBee

### Structure des Paquets NWK

Les paquets ZigBee NWK suivent cette structure (telle que parsée par `ZigBeeNWKPacketParser` dans zigbeedecode.py):

```
+-----------------+----+----+--------+------+------------------+------------------+-----------+--------+
| Frame Control   | DA | SA | Radius | Seq# | Dst IEEE Address | Src IEEE Address | MCast Ctrl| Payload|
| (2 bytes)       |(2) |(2) | (1)    | (1)  | (8 bytes opt.)   | (8 bytes opt.)   | (1 opt.)  |        |
+-----------------+----+----+--------+------+------------------+------------------+-----------+--------+
```

**Frame Control Field Flags**:
- `ZBEE_NWK_FCF_EXT_SOURCE (0x1000)`: Présence de l'adresse source étendue (64-bit)
- `ZBEE_NWK_FCF_EXT_DEST (0x0800)`: Présence de l'adresse destination étendue
- `ZBEE_NWK_FCF_SECURITY (0x0200)`: Sécurité activée (chiffrement)

**Dans les captures analysées**:
- Environ 70% des paquets incluent des adresses étendues
- La sécurité est activée sur ~30% des paquets (non déchiffrés sans clé réseau)

### Méthode de Parsing (KillerBee)

**Code utilisé pour extraire les adresses** (zigbeedecode.py:46-111):

```python
def pktchop(self, packet):
    fc = struct.unpack("<H", packet[0:2])[0]  # Frame Control Field
    pktchop = [packet[0:2], packet[2:4], packet[4:6], packet[6], packet[7]]
    offset = 8

    # Extraction de l'adresse destination étendue si présente
    if (fc & ZBEE_NWK_FCF_EXT_DEST) != 0:
        pktchop.append(packet[offset:offset+8])
        offset += 8

    # Extraction de l'adresse source étendue si présente
    if (fc & ZBEE_NWK_FCF_EXT_SOURCE) != 0:
        pktchop.append(packet[offset:offset+8])
        offset += 8

    # Suite du parsing...
```

---

## Analyse de la Couche Application (APS)

### Structure des Paquets APS

```
+-------------+-------------+---------------+-------------+-------------+-------------+------------+
| Frame Ctrl  | Dst Endpoint| Group Address | Cluster ID  | Profile ID  | Src Endpoint| APS Counter|
| (1 byte)    | (1 opt.)    | (2 opt.)      | (2 bytes)   | (2 bytes)   | (1 opt.)    | (1 byte)   |
+-------------+-------------+---------------+-------------+-------------+-------------+------------+
```

**Modes de Livraison Observés**:
- **Unicast (0x00)**: Majorité des communications Coordinateur ↔ Terminal
- **Broadcast (0x02)**: Utilisé pour les Link Status et Network Status

### Clusters APS Détaillés

#### Cluster 0x0021 - Green Power
- **Utilisation**: Commandes de contrôle Green Power
- **Direction**: Principalement Coordinateur → Terminal
- **Fréquence**: Haute (toutes les 2-3 secondes)
- **Taille**: ~20-25 octets

#### Cluster 0x00a1 - RSSI Location
- **Utilisation**: Transmission des valeurs RSSI pour la localisation
- **Direction**: Principalement Terminal → Coordinateur
- **Fréquence**: Réponse aux requêtes du coordinateur
- **Taille**: ~18-22 octets

---

## Outils et Commandes Utilisés pour l'Analyse

### 1. Analyse avec tshark (Wireshark CLI)

```bash
# Extraction des adresses IEEE 64-bit
tshark -r coordinateur.pcapng -T fields \
    -e zbee_nwk.src64 -e zbee_nwk.dst64 \
    -e wpan.src16 -e wpan.dst16

# Filtrage par adresse source
tshark -r coordinateur.pcapng \
    -Y "zbee_nwk.src64 == 00:13:a2:00:42:34:63:55"

# Extraction des commandes NWK
tshark -r terminal.pcapng -Y "zbee_nwk.cmd.id" \
    -T fields -e frame.number -e zbee_nwk.cmd.id \
    -e zbee_nwk.src64

# Statistiques globales
tshark -r coordinateur.pcapng -q -z io,stat,0
```

### 2. Analyse avec KillerBee

```bash
# Affichage des paquets capturés
zbcat -f coordinateur.pcapng

# Lecture avec décodage ZigBee
zbdump -f capture.pcap -c 11

# Conversion pour Daintree SNA
zbconvert coordinateur.pcapng coordinateur.dcf
```

### 3. Filtres Wireshark Utilisés

```
# Filtrer le coordinateur comme source
zbee_nwk.src64 == 00:13:a2:00:42:34:63:55

# Filtrer le terminal comme source
zbee_nwk.src64 == 00:13:a2:00:41:fb:9b:ee

# Communications entre les deux dispositifs
(zbee_nwk.src64 == 00:13:a2:00:42:34:63:55 &&
 zbee_nwk.dst64 == 00:13:a2:00:41:fb:9b:ee) ||
(zbee_nwk.src64 == 00:13:a2:00:41:fb:9b:ee &&
 zbee_nwk.dst64 == 00:13:a2:00:42:34:63:55)

# Commandes NWK uniquement
zbee_nwk.cmd.id

# Cluster Green Power
zbee_aps.cluster == 0x0021

# Paquets de type Data seulement
wpan.frame_type == 0x0001
```

---

## Classes et Méthodes KillerBee Utilisées

### ZigBeeNWKPacketParser (zigbeedecode.py)

**Méthodes principales**:

1. **pktchop(packet)**
   - Parse les en-têtes NWK en liste de champs
   - Extrait les adresses étendues si présentes
   - Retourne: `[FCF, DA, SA, Radius, Seq#, DstIEEE, SrcIEEE, MCast, Payload]`

2. **hdrlen(packet)**
   - Calcule la longueur de l'en-tête NWK
   - Prend en compte les adresses étendues et le source routing

3. **payloadlen(packet)**
   - Retourne la taille du payload NWK (données APS)

### Dot154PacketParser (dot154decode.py)

**Méthodes principales**:

1. **pktchop(packet)**
   - Parse les en-têtes IEEE 802.15.4
   - Retourne: `[FCF, Seq#, DPAN, DA, SPAN, SA, BeaconData, Payload]`

2. **decrypt(packet, key)**
   - Déchiffre les paquets avec AES-CCM
   - Vérifie le MIC (Message Integrity Code)

3. **nonce(packet)**
   - Extrait le nonce pour le déchiffrement
   - Format: `Src Addr || Frame Counter || Security Level`

### ZigBeeAPSPacketParser (zigbeedecode.py)

**Méthodes principales**:

1. **pktchop(packet)**
   - Parse les en-têtes APS
   - Gère les différents modes de livraison (Unicast, Broadcast, Group)
   - Retourne: `[FCF, DstEP, GroupAddr, ClusterID, ProfileID, SrcEP, Counter, Payload]`

---

## Informations de Sécurité

### Observations

1. **Chiffrement**: Présence de paquets chiffrés (Security Level 0x05 - AES-CCM avec MIC 32-bit)
2. **Clé réseau**: Non fournie dans la capture (nécessaire pour déchiffrement)
3. **Vulnérabilités potentielles**:
   - Présence d'adresses en clair permettant le tracking des dispositifs
   - Possible de rejouer les commandes NWK Leave (0x02)
   - Absence de chiffrement sur certains paquets de contrôle

### Méthodes d'Extraction de Clés (KillerBee)

```bash
# Sniff de clés lors du provisioning OTA
zbdsniff -f capture.pcap

# Recherche de clés dans un dump mémoire
zbgoodfind -f memory.bin -t

# Tentative de déchiffrement avec clé connue
# (via scapy_extensions.py - kbdecrypt)
```

---

## Topologie du Réseau Identifiée

```
    [Coordinateur]              [Router]              [Terminal]
 00:13:a2:00:42:34:63:55  00:13:a2:00:41:fb:76:ea  00:13:a2:00:41:fb:9b:ee
   Adresse: 0x0000           Adresse: 0x6b82         Adresse: 0xc7f4
         |                         |                        |
         +----------> RELAIS <-----+------------------------+
```

**Caractéristiques**:
- **Type de réseau**: Maillé (Mesh) avec routage multi-sauts
- **Communication**: Le router (0x6b82) relaie TOUS les paquets entre coordinateur et terminal
- **Routage**: AODV simplifié (AODVjr) avec flag Discovery Enable
- **PAN ID**: Non spécifié dans l'analyse (à extraire avec zbdump)
- **Canal**: Non spécifié (probablement 11-26, bande 2.4 GHz)
- **Protocole**: ZigBee Green Power (Profile 0xc105)

---

## Statistiques Détaillées

### Coordinateur (44.7 secondes de capture)
- **Total paquets**: 2168
- **Taux moyen**: 48.5 paquets/seconde
- **Bande passante**: 1.36 KB/s
- **ACK ratio**: 49.0%
- **Paquets de données**: 18.5%
- **Commandes NWK**: 0.7%

### Terminal (251.9 secondes de capture)
- **Total paquets**: 16688
- **Taux moyen**: 66.2 paquets/seconde
- **Bande passante**: 1.89 KB/s
- **ACK ratio**: 47.7%
- **Paquets de données**: 14.3%
- **Commandes NWK**: 1.1%

### Observations
- Le terminal génère plus de trafic que le coordinateur (ratio 7.7:1)
- Le taux d'ACK est similaire sur les deux dispositifs (~48%)
- La capture du terminal est 5.6× plus longue que celle du coordinateur

---

## Conclusion

### Méthodes d'Analyse Employées

1. **Filtrage par adresse étendue (64-bit)**:
   - `zbee_nwk.src64` pour identifier la source
   - `zbee_nwk.dst64` pour identifier la destination

2. **Extraction des en-têtes avec ZigBeeNWKPacketParser**:
   - Parsing du Frame Control Field
   - Extraction conditionnelle des adresses étendues
   - Calcul des longueurs de payload

3. **Analyse des clusters APS**:
   - Identification des profils (Green Power 0xc105)
   - Décodage des clusters (0x0021, 0x00a1)

4. **Analyse des commandes NWK**:
   - Network Status (0x01)
   - Leave (0x02)
   - Link Status (0x08)

5. **Corrélation adresses courtes/étendues**:
   - Analyse des paquets avec `wpan.src16` et `zbee_nwk.src64`
   - Construction de la table d'adressage

### Informations Clés Extraites

- **Coordinateur**: 00:13:a2:00:42:34:63:55 (0x0000)
- **Terminal principal**: 00:13:a2:00:41:fb:9b:ee (0xc7f4)
- **Profil utilisé**: Green Power (0xc105)
- **Communication**: Bidirectionnelle avec ACK systématiques
- **Topologie**: Réseau en étoile avec coordinateur central

### Applications Pratiques

Cette analyse permet de:
- Identifier tous les dispositifs du réseau ZigBee
- Comprendre les patterns de communication
- Détecter des anomalies ou attaques potentielles
- Préparer des attaques ciblées (replay, injection, flooding)
- Extraire les clés réseau lors de provisioning

---

## Références

### Fichiers KillerBee Consultés
- `killerbee/zigbeedecode.py`: ZigBeeNWKPacketParser, ZigBeeAPSPacketParser
- `killerbee/dot154decode.py`: Dot154PacketParser, déchiffrement AES-CCM
- `killerbee/scapy_extensions.py`: Intégration Scapy, kbdecrypt
- `tools/zbdump`: Capture de paquets
- `tools/zbcat`: Affichage de paquets
- `tools/zbdsniff`: Extraction de clés réseau

### Standards
- IEEE 802.15.4 (couche PHY/MAC)
- ZigBee 3.0 Specification (couche NWK/APS)
- ZigBee Green Power Specification

---

**Analyse générée le**: 2025-12-15
**Outils utilisés**: KillerBee, Wireshark/tshark, Python
**Fichiers analysés**: coordinateur.pcapng, terminal.pcapng
