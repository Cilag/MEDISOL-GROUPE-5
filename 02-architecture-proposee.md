# 02 — Architecture proposée

## 2.1 Principes directeurs

L'architecture retenue repose sur six axes :

1. **Élimination du SPOF** — deux serveurs physiques avec réplication automatique des VMs
2. **Virtualisation type 1** — Proxmox VE sur bare-metal, isolation des services critiques en VMs dédiées
3. **Segmentation réseau stricte** — VLANs par zone fonctionnelle, pare-feu OPNsense inter-zones
4. **Wi-Fi managé** — contrôleur Wi-Fi logiciel avec SSIDs distincts, QoS et isolation des patients
5. **Conformité RGPD** — chiffrement au repos et en transit, journalisation des accès aux données patients
6. **Budget large disponible** — l'absence de contrainte financière permet d'opter pour des solutions robustes et redondantes (cluster 3 nœuds si justifié, SAN dédié, sauvegarde professionnelle)

---

## 2.2 Vue d'ensemble de l'architecture cible


<img src="assets/medisol_archi.drawio.svg">
---

## 2.3 Plan d'adressage et VLANs

| VLAN | Nom | Plage IP | Accès autorisé | Accès interdit |
|---|---|---|---|---|
| **VLAN 10** | User | 10.10.0.0/24 | VLAN 30, imprimantes | VLAN30/40/50/99 |
| **VLAN 20** | Clients | 10.20.0.0/24 | VLAN 30, imprimantes | VLAN30/40/50/99 |
| **VLAN 30** | Infra | 10.30.0.0/24 | DNS, AD/LDAP, DHCP | Tout accès direct externe |
| **VLAN 40** | IoT | 10.40.0.0/24 | VLAN 30 | Tous les autres VLANs |
| **VLAN 50** | Sécurité | 10.50.0.0/24 | NVR local uniquement | Tous les autres VLANs |
| **VLAN 99** | Admin | 10.99.0.0/24 | Accès INFRA, IOT & Security | Tous les autres VLANs internes |

---

## 2.4 Composants de l'architecture

### Serveurs physiques

| Composant | Spécification recommandée |
|---|---|
| SRV1 (primaire) | Dell PowerEdge R350 — 2x Xeon Silver, 64 GB RAM, 2x 1 TB NVMe + 4x 4 TB SAS |
| VEEAM BackUp SRV | 4 vCPU, 16–32 Go RAM |
| Stockage ZFS | RAID Z1 sur SRV1 — réplication ZFS bidirectionnelle |
| NAS | NAS — 4-8 To+ par disque selon besoin |


### Machines virtuelles

| VM | Rôle | OS | vCPU | RAM | Stockage |
|---|---|---|---|---|---|
| VM-VEEAM | Serveur de Backup | Windows Server 2025 | 8 | 16 GB | 500 GB |
| VM-IMAGERIE | Stockage imagerie légère (SMB + DFS) | Windows Server 2025 | 4 | 8 GB | 2 TB |
| VM-MON | Supervision (Prometheus + Grafana) | Ubuntu 26.04 | 2 | 4 GB | 100 GB |
| VM-WEB | Portail patient (Nginx + app) | Ubuntu 26.04 | 2 | 4 GB | 50 GB |

> **Dimensionnement stockage RGPD** : le volume de données actuel est de **5 TB** avec une croissance de **10 GB/mois** (~120 GB/an). À 5 ans, le volume atteindra ~5,6 TB de données actives. Les recommandations RGPD (durée de conservation, chiffrement, journalisation) imposent une marge supplémentaire. Le cluster est dimensionné avec 4x 4 TB SAS par nœud (ZFS RAID-Z1 = ~8 TB utile/nœud) pour absorber 5+ ans de croissance.

### Réseau

- **Pare-feu** : Sophos 
- **Switch** : Switch manageable L2/L3 24 ports PoE 
- **Wi-Fi** : 2x Access Points Wi-Fi 6 (Ubiquiti UniFi U6-Pro ou équivalent)
  - SSID `MEDISOL-User` → VLAN 10 (utilisateurs de l'entreprise) 
  - SSID `MEDISOL-Clients` → VLAN 20 (isolé, portail captif)

---

## 2.5 Politique de sécurité réseau (Sophos)

```
# Extrait règles
VLAN10 -> VM-PATIENT (10.30.x.x) : TCP/3389 ALLOW (log)
VLAN10 -> VM-FILE (10.30.y.y) : TCP/445 ALLOW (si nécessaire)
VLAN10 -> VLAN99 : DENY
VLAN20 -> VLAN30 : TCP/443, TCP/22, TCP/3389, TCP/445 ALLOW (hosts précis)
VLAN20 -> VLAN99 : DENY
VLAN50 (caméras/NVR intra) : RTSP/TCP 554, RTP/UDP range, TCP/443 ALLOW (intra-VLAN)
VLAN50 (NVR) -> NAS (10.30.z.z) : TCP/445,TCP/2049,TCP/3260 ALLOW (NVR IP only)
VLAN40 -> VLAN30 : TCP/8883, TCP/443, UDP/TCP/53, UDP/123 ALLOW
VLAN40 -> others : DENY
VLAN99 -> Internet : TCP/80,443, UDP/TCP/53 ALLOW ; interne DENY
OpenVPN_clients -> VLAN10 (VM-PATIENT 10.10.0.x) : TCP/3389, TCP/443 ALLOW (log, restrict to user group)
OpenVPN_clients -> others : DENY
VEEAM_IP -> VLAN 30 

```

---

## 2.6 Haute disponibilité et réplication

- **Proxmox HA Manager** : bascule automatique des VMs critiques (VM-PATIENT, VM-IMAGERIE) en cas de panne nœud
- **ZFS Replication** : réplication synchrone des datastores (toutes les 15 min)
- **VEEAM** : sauvegardes nightly 23h00, rétention 30 jours

---

## 2.7 Accès nomades — praticiens itinérants

```
Praticien (domicile / cabinet secondaire)
    │
    └─[VPNSSL]──► Sophos Firewall
                              │
                              └──► VLAN 10
                                    │
                                    └──► VM-PATIENT (RDS RemoteApp)
                                         └──► Logiciel patient
```

- Authentification : certificat OpenVPN + MFA Entra ID (Authenticator)
- Accès limité à VM-PATIENT uniquement (VLAN 10, port TCP 3389)
- Aucun accès aux VLANs Admin, IoT ou Serveurs

---

## 2.8 Portail patient — DMZ

Le portail patient (en développement) est hébergé dans **VM-WEB** (VLAN 30, sous-zone DMZ) :

```
Internet ──► VPNSSL (HTTPS 443) ──► VM-WEB (Nginx reverse proxy)
                                         │
                                         └──► Application portail patient
                                              └──► Base de données chiffrée
```

- Flux sortant limité : VM-WEB ne peut pas accéder aux VLANs 10/20/40
- Certificat TLS automatique
- WAF Sophos pour protection des applications web
