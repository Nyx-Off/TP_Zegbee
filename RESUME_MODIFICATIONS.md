# Résumé des Modifications - KillerBee sur Python 3.13

## Contexte

KillerBee est un framework pour tester et auditer les réseaux ZigBee et IEEE 802.15.4. Le projet nécessitait des corrections pour fonctionner correctement sur Python 3.13+ sous Kali Linux.

## Problèmes Identifiés

1. **Dépendance obsolète**: `pycrypto` ne compile pas sur Python 3.13+
2. **Avertissement syntaxe**: Utilisation incorrecte de `is not` pour comparer des entiers
3. **Documentation manquante**: Absence de guide pour l'installation moderne

## Modifications Apportées

### 1. Migration cryptographique (setup.py)

**Fichier**: `setup.py:81`

**Avant**:
```python
install_requires=['pyserial>=2.0', 'pyusb', 'pycrypto', 'rangeparser', 'scapy']
```

**Après**:
```python
install_requires=['pyserial>=2.0', 'pyusb', 'pycryptodome', 'rangeparser', 'scapy']
```

**Raison**: `pycrypto` est obsolète et ne compile pas sur Python 3.13+. `pycryptodome` est API-compatible et maintenu activement.

### 2. Correction syntaxe Python (dev_cc253x.py)

**Fichier**: `killerbee/dev_cc253x.py:211`

**Avant**:
```python
if e.errno is not 110 and e.errno is not 60: #Operation timed out
```

**Après**:
```python
if e.errno != 110 and e.errno != 60: #Operation timed out
```

**Raison**: L'opérateur `is/is not` compare l'identité des objets, pas leur valeur. Pour les entiers, il faut utiliser `==/!=`. Cela élimine les warnings Python et suit les meilleures pratiques.

### 3. Compilation extension C

**Fichier généré**: `zigbee_crypt.cpython-313-x86_64-linux-gnu.so`

Extension C compilée pour Python 3.13 fournissant les fonctions cryptographiques ZigBee (AES-CCM) via libgcrypt.

### 4. Documentation (CLAUDE.md)

Création d'un guide complet incluant:
- Instructions d'installation pour Kali/Debian (avec `--break-system-packages` pour PEP 668)
- Configuration des permissions USB (udev rules)
- Commandes courantes (capture, attaques, récupération de clés)
- Architecture du code (drivers, modules, outils)
- Patterns de développement et pièges courants

## Installation Vérifiée

```bash
# Dépendances système
sudo apt-get install python3-usb python3-serial python3-dev libgcrypt-dev

# Installation en mode développement (requis sur Kali)
sudo pip3 install -e . --break-system-packages

# Vérification
sudo zbid  # Liste les périphériques détectés
```

## Tests Effectués

- Compilation réussie de l'extension C `zigbee_crypt`
- Installation des dépendances Python sans erreur
- Commande `zbid` fonctionnelle pour détecter les dongles USB CC2531

## Fichiers Modifiés

```
M killerbee/dev_cc253x.py      # Correction syntaxe
M setup.py                      # Migration pycryptodome
A CLAUDE.md                     # Documentation
A zigbee_crypt.*.so            # Extension compilée
```

## Compatibilité

- Python 3.13+ (testé)
- Kali Linux (avec PEP 668)
- Hardware supporté: CC2530/CC2531, ApiMote, RZ RAVEN USB Stick, etc.

## Notes Importantes

1. Sur Kali, l'option `--break-system-packages` est nécessaire à cause de PEP 668
2. Permissions USB requises: soit `sudo`, soit configuration udev rules
3. La branche `develop` est la branche de développement active (pas `master`)
