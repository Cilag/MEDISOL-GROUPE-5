# 04 — Objectifs pédagogiques

> Ce document décline les 8 objectifs officiels du MSPR Virtualisation, appliqués concrètement au cas MEDISOL.

---

## Objectif 1 — Proxmox — hyperviseur type 1 adapté au besoin

### Pourquoi Proxmox VE pour MEDISOL ?

Proxmox VE est un hyperviseur de **type 1** (bare-metal) basé sur Debian Linux avec KVM comme moteur de virtualisation. Il s'installe directement sur le matériel sans OS hôte intermédiaire, garantissant des performances maximales et une isolation robuste entre les VMs.

### Adéquation au besoin MEDISOL

| Critère MEDISOL | Réponse Proxmox VE |
|---|---|
| Aucune IT interne | Interface d'administration web (port 8006), administrable par un prestataire externe |
| Budget contraint | Licence AGPL-3.0 — gratuit en production ; support optionnel (~180 €/an/nœud) |
| Zéro panne exigé | Cluster HA natif sur 2 nœuds : bascule automatique en < 5 min |
| Données patients sensibles | Isolation VM + ZFS chiffré, journalisation Proxmox complète |
| Petite structure | 2 nœuds suffisent ; évolutif si besoin (3e nœud possible) |

### Comparaison avec XCP-ng

XCP-ng (fork de XenServer) est une alternative type 1 valide :
- Hyperviseur Xen vs KVM pour Proxmox — les deux sont matures
- XCP-ng dispose de Xen Orchestra (XO) pour l'administration web
- **Proxmox est retenu** : HA plus simple à configurer pour une petite structure ; la communauté francophone est plus active

### Lien avec le cours

Ce cas illustre le choix d'un hyperviseur en fonction de contraintes organisationnelles (absence d'IT), réglementaires (RGPD), et techniques (HA sur 2 nœuds dans un petit local technique).

---

## Objectif 2 — Ressources & sécurité — dimensionnement, isolation, accès, tiers

### Dimensionnement des ressources

Basé sur 32 utilisateurs concurrents max (dont ~20 en pointe) :

| VM | vCPU | RAM | Stockage | Justification |
|---|---|---|---|---|
| VM-PATIENT | 8 | 16 GB | 150 GB | RDS : jusqu'à 10 sessions RemoteApp simultanées |
| VM-IMAGERIE | 4 | 8 GB | 2 TB | I/O stockage imagerie légère (accès séquentiels) |
| VM-MON | 2 | 4 GB | 100 GB | Supervision continue, rétention métriques 90 j |
| VM-WEB | 2 | 4 GB | 50 GB | Portail patient — pic trafic matin (prises de RDV) |
| VEEAM | 4 | 8 GB | 4 TB | Sauvegarde déduplicatée — 6 VMs × 30 jours |
| Sophos | 2 | 2 GB | 20 GB | Routage inter-VLAN + IDS Suricata |

**Overcommit** : 1:1.5 CPU, 1:1.2 RAM — raisonnable car les VMs Windows et Debian ne piquent pas simultanément.

### Isolation et sécurité

- **Isolation réseau VM** : chaque VM sur son VLAN propre — pas de communication directe inter-VM sans passer par Sophos
- **Isolation stockage** : chaque VM sur son propre dataset ZFS chiffré (ZFS native encryption)
- **ACLs Proxmox** :
  - `PVEAdmin` : prestataire IT (accès complet)
  - `PVEAudit` : responsable administratif MEDISOL (lecture seule)
  - `PVEVMUser` : opérateur astreinte (démarrage/arrêt VMs uniquement)

### Niveaux de service (tiers)

| Tier | VMs | Disponibilité cible | Protection |
|---|---|---|---|
| Critique (Tier 1) | VM-PATIENT, VM-IMAGERIE | 99,9 % | HA Proxmox + réplication ZFS |
| Important (Tier 2) | VM-WEB, Sophos | 99,5 % | Réplication VEEAM nightly |
| Standard (Tier 3) | VM-MON, VM-VEEAM | 99 % | Backup hebdomadaire |

---

## Objectif 3 — Hybride / SaaS 365 / local / hébergement backup externe

### Stratégie hybride MEDISOL

MEDISOL utilise déjà Microsoft 365. L'architecture adopte une position **hybride pragmatique** :

| Couche | Solution | Hébergement | Raison |
|---|---|---|---|
| Logiciel patient | Client lourd RDS | On-prem (VM-PATIENT) | Contrainte éditeur — pas de version cloud |
| Imagerie légère | Windows Server SMB | On-prem (VM-IMAGERIE) | Volumes > 500 GB, accès LAN rapide requis |
| Messagerie / Collab | Microsoft 365 | Cloud (SaaS) | Déjà déployé, disponibilité garantie |
| Prise de RDV en ligne | Portail existant | Cloud ou On-prem selon éditeur | Continuité de service indépendante |
| Portail patient | Application web | On-prem DMZ (VM-WEB) | Données patients — préférence hébergement local |
| Sauvegarde | VEEAM + Backup externe | Hybride | Règle 3-2-1 : local rapide + hébergeur hors site |
| Identité / MFA | Entra ID | Cloud (SaaS M365) | MFA nomades + SSO |

### Entra ID — MFA pour les praticiens nomades

```
Praticien (nomade) ──[Authenticator MFA]──► Entra ID ──► OpenVPN Activation
                                                │
                                           Conditional Access
                                           (périmètre géographique)
```

La politique Conditional Access bloque les connexions hors France et hors horaires ouvrés.

### Position sur le portail patient

Le portail patient est hébergé **on-prem** dans une DMZ pour garder les données de santé dans un périmètre maîtrisé, mais l'accès public se fait via IP dédiée avec TLS Let's Encrypt géré automatiquement.

---

## Objectif 4 — Supervision — suivi des VMs / services critiques

### Architecture de supervision

VM-MON (Ubuntu 26.04) centralisé :

```
Netdata Agent (chaque VM)
        │
        ▼
Prometheus scrape (toutes les 30s)
        │
        ▼
Grafana (dashboards + alertes)
        │
        ├──► Email (prestataire IT) — toutes alertes
        └──► Teams webhook — alertes critiques uniquement
```

### Métriques surveillées

| Objet | Métriques clés | Seuil alerte | Seuil critique |
|---|---|---|---|
| Nœuds Proxmox (SRV1/2) | CPU, RAM, I/O disque, réseau | 75 % | 90 % |
| VM-PATIENT | Sessions RDS actives, CPU, RAM, latence app | > 20 sessions | 25 sessions |
| VM-IMAGERIE | Espace disque, IOPS, sessions SMB | 80 % disque | 95 % disque |
| VM-WEB | HTTP response time, sessions actives | > 3 s | HTTP 5xx |
| VEEAM | Dernier backup (timestamp), taux dédup | > 26h sans backup | Job échoué |
| Sophos | Bande passante WAN, état WireGuard, IDS alerts | 80 % BW | Tunnel VPN down |
| Wi-Fi | Clients connectés / SSID, signal qualité | > 50 clients VLAN 99 | AP unreachable |

### Alertes conformité RGPD

- Accès hors horaires (23h-7h) à VM-PATIENT → alerte immédiate prestataire
- Tentatives d'accès VLAN 20 - Clients → ALL VLAN bloquées par Sophos → log + alerte

---

## Objectif 5 — Sauvegardes & PRA — stratégie, tests de restauration

### Politique de sauvegarde (règle 3-2-1)

> **3** copies · **2** supports différents · **1** copie hors site

| Copie | Emplacement | Technologie | Fréquence | Rétention |
|---|---|---|---|---|
| **Copie 1** (production) | SRV1 — ZFS live | ZFS snapshots | Toutes les 4h | 7 jours |
| **Copie 2** (locale) | SRV2 — VEEAM | Proxmox Backup Server | Nightly 23h00 | 30 jours |
| **Copie 3** (hors site) | Hébergeur | Solution professionelle | Mar + Ven 02h00 | 12 mois |

### RTO / RPO

| Scénario | RTO | RPO |
|---|---|---|
| Corruption logicielle d'une VM | < 30 min | < 4h (snapshot ZFS) |
| Panne d'un nœud Proxmox | < 5 min | 0 (HA bascule automatique) |
| Panne des 2 nœuds | < 4 h | < 24h (restauration PBS) |
| Sinistre total (incendie) | < 8 h | < 48h (Azure Backup) |

### Plan de tests de restauration

| Test | Fréquence | Procédure | Succès |
|---|---|---|---|
| Restauration VM depuis VEEAM | Mensuel | Clone dans env. isolé Proxmox | VM démarre, données cohérentes |
| Vérification Backup | Mensuel | Restauration fichier unique | Fichier récupéré < 30 min |
| Test bascule HA | Trimestriel | Éteindre SRV1, vérifier migration VMs | VMs opérationnelles < 5 min |
| Simulation PRA complet | Annuel | Restauration depuis hébergeur sur matériel test | Logiciel patient fonctionnel < 4h |

---

## Objectif 6 — VDI & profils — accès distant / postes centralisés si pertinent

### Analyse du besoin VDI chez MEDISOL

Une infrastructure VDI complète est **disproportionnée** pour MEDISOL à ce stade. La solution retenue est un compromis efficace :

| Population | Effectif | Solution retenue | Justification |
|---|---|---|---|
| Accueil (postes partagés) | 8 | **RDS RemoteApp** depuis VM-PATIENT | Postes partagés → profils AD itinérants |
| Praticiens nomades | ~10 | **RDS via OpenVPN Sophos** | Accès sécurisé hors site, même technologie |
| Administration | 12 | Postes fixes VLAN 20 + M365 | Peu de besoins de mobilité |
| Équipements mesure | - | VLAN 40 dédié IoT | Aucune virtualisation requise |

### RDS RemoteApp — postes d'accueil partagés

```
Poste accueil (Windows 11, joint au domaine)
    │
    └─[RemoteApp]──► VM-PATIENT (RDS Server)
                         │
                         └──► Logiciel patient (session isolée par utilisateur)
                              └──► Profil AD itinérant (données sur VM-IMAGERIE)
```

**Avantages** :
- Mise à jour du logiciel patient centralisée sur VM-PATIENT uniquement
- Sessions isolées entre utilisateurs (sécurité RGPD)
- Postes d'accueil remplacés par des thin clients si besoin

### Scénario VDI futur

Si MEDISOL ouvre un cabinet secondaire, Proxmox VE supporte nativement les sessions VDI (SPICE/RDP) sans coût de licence supplémentaire.

---

## Objectif 7 — PRA / PCO — plan de continuité documenté

### Scénarios couverts

#### Scénario A — Panne d'un nœud Proxmox

**Détection** : alerte VM-MON en < 5 min  
**Impact** : VMs basculent automatiquement sur le nœud survivant (HA)  
**Actions** :
1. Confirmer la bascule HA depuis l'interface Proxmox
2. Alerter le prestataire IT (SLA 4h ouvrées)
3. Diagnostiquer et commander pièces si matériel défaillant
4. Réintégrer le nœud dans le cluster après réparation  
**RTO** : < 5 min automatique

#### Scénario B — Corruption du logiciel patient (mise à jour ratée)

**Détection** : retour d'erreur des utilisateurs  
**Actions** :
1. Revenir à un snapshot ZFS de VM-PATIENT (avant mise à jour)
2. Durée estimée : < 15 min  
**RTO** : < 30 min | **RPO** : < 4h

#### Scénario C — Ransomware

**Détection** : activité I/O anormale sur VM-IMAGERIE, alertes IDS  
**Actions** :
1. Isoler le(s) VLAN(s) suspects (Sophos — couper les règles)
2. Identifier l'origine (logs Sophos, logs VM)
3. Vérifier l'intégrité des backups PBS (immutables)
4. Restaurer depuis PBS ou Azure Backup selon périmètre  
**RTO** : < 4h | **RPO** : < 24h

#### Scénario D — Sinistre total (incendie)

**Actions** :
1. Activer le mode dégradé : M365 disponible en cloud pour messagerie et Teams
2. Contacter le prestataire IT pour déploiement d'urgence
3. Restaurer les VMs critiques depuis Azure Backup Vault
4. Priorité : VM-PATIENT (logiciel patient) et VM-WEB (prise de RDV)  
**RTO** : < 8h | **RPO** : < 48h

### Annuaire de crise

| Rôle | Contact | Disponibilité |
|---|---|---|
| Responsable décision (direction MEDISOL) | [Tél. direct] | H ouvrées |
| Prestataire IT principal | [Tél. astreinte] | H24 |
| Support hébergeur | Portail hébergeur | H24 |
| Éditeur logiciel patient | [Support éditeur] | H ouvrées |
| FAI | [Numéro astreinte] | H24 |
