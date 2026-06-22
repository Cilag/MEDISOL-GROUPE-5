# 03 — Mise en œuvre

## 3.1 Phase 1 — Déploiement de l'hyperviseur Proxmox VE

### Installation SRV1 (nœud primaire)

```bash
# Installation Proxmox VE 8.x depuis ISO sur clé USB
# Paramètres d'installation :
# - Hostname : pve-medisol-01.local
# - IP management : 10.30.0.11/24
# - Gateway : 10.30.0.1 (Sophos)
# - DNS : 10.0.30.1

# Post-installation — désactiver le repo entreprise, activer no-subscription
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list
apt update && apt dist-upgrade -y
```

### Installation SRV2 (nœud secondaire)

- Même procédure : hostname `pve-medisol-02.local`, IP `10.30.0.12/24`

### Création du cluster Proxmox

```bash
# Sur SRV1 — créer le cluster
pvecm create medisol-cluster

# Sur SRV2 — rejoindre le cluster
pvecm add 10.30.0.11
```

### Configuration ZFS

```bash
# Sur chaque nœud : créer le pool ZFS pour les VMs
zpool create -f vmdata raidz /dev/sdb /dev/sdc /dev/sdd /dev/sde

# Activer la compression
zfs set compression=lz4 vmdata

# Ajouter le stockage dans Proxmox
pvesm add zfspool vmdata-pool --pool vmdata --content images,rootdir
```

---

## 3.3 Phase 2 — Déploiement Sophos (pare-feu)

### Création de la VM Sophos

```
Ressources : 2 vCPU, 2 GB RAM, 20 GB disque
Interfaces réseau :
  - vmbr0 : WAN (vers box FAI)
  - vmbr1 : LAN (vers LAN Proxmox)
  - vmbr1.10 : VLAN 10 (USERS)          - 10.10.0.1/24
  - vmbr1.20 : VLAN 20 (Clients)        - 10.20.0.1/24
  - vmbr1.30 : VLAN 30 (INFRA)          - 10.30.0.1/24
  - vmbr1.40 : VLAN 40 (IoT)            - 10.40.0.1/24
  - vmbr1.50 : VLAN 50 (Security)       - 10.50.0.1/24
  - vmbr1.99 : VLAN 99 (Admin)          - 10.99.0.1/24
```

### Règles de filtrage inter-VLAN

Configuration via l'interface OPNsense → Firewall → Rules :

| Source | Destination | Port | Action | Description |
|---|---|---|---|---|
| VLAN10 net | VLAN30 net | TCP 445, 3389 | Pass | Accueil → logiciel patient + imagerie |
| VLAN10 net | WAN | TCP 443, 80 | Pass | Accueil → M365 |
| VLAN20 net | VLAN30 net | TCP 445, 443 | Pass | Admin → serveurs |
| VLAN20 net | WAN | TCP 443, 80 | Pass | Admin → M365 |
| VLAN40 net | !VLAN40 net | * | Block | IoT isolé |
| VLAN99 net | WAN | TCP 443, 80 | Pass | Patients → Internet uniquement |
| VLAN99 net | !WAN | * | Block | Isolation patients |
| WireGuard | VLAN10 net | TCP 3389 | Pass | Nomades → RDS |

---

## 3.4 Phase 3 — Déploiement des VMs

### VM-PATIENT — Logiciel patient 


### VM-IMAGERIE — Stockage imagerie légère


### VM-MON — Supervision


### VM-WEB — Portail patient (DMZ)


---

## 3.5 Phase 4 — Infrastructure Wi-Fi managée

### Déploiement des Access Points

```
AP1 (Salle d'attente / Accueil) — PoE sur port switch VLAN trunk
AP2 (Zone cabinets)              — PoE sur port switch VLAN trunk

SSIDs configurés :
  MEDISOL-Praticiens  → VLAN 10 (WPA3-Enterprise, auth M365)
  MEDISOL-Clients     → VLAN 20 (WPA3-Personal, portail captif)
```

### Configuration portail captif patients (VLAN 20)

- Portail captif Sophos sur VLAN 20 : acceptation CGU avant accès Internet
- QoS : bande passante patients limitée à 10 Mb/s (up/down) pour ne pas saturer le lien pro
- Isolation client-à-client : les patients ne peuvent pas communiquer entre eux sur le Wi-Fi

---

## 3.6 Phase 5 — VPN nomades (praticiens itinérants)

- Chaque praticien reçoit un profil Sophos (.ovpn)
- Authentification renforcée : MFA Entra ID requis avant activation du tunnel (via Conditional Access)
- Accès limité à VM-PATIENT (10.30.0.6, port 3389) — règle Sophos dédiée

---

## 3.7 Phase 6 — Sauvegardes (VEEAM)

### Windows Server avec client VEEAM


---

## 3.8 Migration des données imagerie existantes

```bash
# Depuis le serveur Windows Server existant vers VM-IMAGERIE
# Utiliser robocopy pour migration sans perte

robocopy \\ancienServeur\Imagerie \\vm-imagerie\Imagerie /MIR /COPYALL /LOG:migration.log

# Vérification intégrité après migration
# Comparer les checksums MD5 des fichiers source et destination
Get-ChildItem -Recurse \\ancienServeur\Imagerie |
    Get-FileHash -Algorithm MD5 | Export-Csv checksums_source.csv

Get-ChildItem -Recurse \\vm-imagerie\Imagerie |
    Get-FileHash -Algorithm MD5 | Export-Csv checksums_destination.csv
```
