# MEDISOL — Dossier MSPR Virtualisation

## Contexte client (résumé)

MEDISOL est un centre de consultations et d'imagerie légère (bien-être / médecine douce) comptant 32 personnes (accueil, praticiens, administration). Le planning est serré et la direction exige une disponibilité totale du système d'information. Les dossiers patients sont soumis à une exigence forte de confidentialité. Le Wi-Fi est saturé par le mélange patients/personnel, et il n'existe aucune séparation entre réseaux invités, métier et administration. Le logiciel métier patient (client lourd Windows) est vieillissant et ralentit aux heures de pointe.

## Index des fichiers

| Fichier | Contenu |
|---|---|
| [01-contexte-besoin.md](01-contexte-besoin.md) | Analyse du besoin : contraintes techniques, objectifs direction, contraintes budgétaires |
| [02-architecture-proposee.md](02-architecture-proposee.md) | Architecture virtualisée cible avec schémas Mermaid (topologie, réseau, flux) |
| [03-mise-en-oeuvre.md](03-mise-en-oeuvre.md) | Mise en œuvre : hyperviseur Proxmox VE, configuration VMs, réseau, stockage |
| [04-objectifs-pedagogiques.md](04-objectifs-pedagogiques.md) | Les 8 objectifs pédagogiques officiels du MSPR appliqués au cas MEDISOL |
| [05-evolutions-entretien-2.md](05-evolutions-entretien-2.md) | Pistes d'évolution et points de discussion — préparation entretien 2 |
| [assets/](assets/) | Répertoire pour schémas supplémentaires |

## Arborescence

```
medisol/
├── README.md                     ← Ce fichier
├── 01-contexte-besoin.md         ← Analyse du besoin
├── 02-architecture-proposee.md  ← Architecture cible + schémas
├── 03-mise-en-oeuvre.md          ← Implémentation technique
├── 04-objectifs-pedagogiques.md  ← 8 objectifs MSPR
├── 05-evolutions-entretien-2.md  ← Évolutions / entretien 2
└── assets/                       ← Ressources graphiques
```

## Points clés de la solution

- **Hyperviseur** : Proxmox VE 8.x (type 1, open-source) sur 2 serveurs physiques
- **Confidentialité** : segmentation VLAN stricte (invités, métier, imagerie, administration) + pare-feu OPNsense
- **Wi-Fi** : contrôleur Wi-Fi avec SSIDs distincts par VLAN (patients, praticiens, back-office)
- **Logiciel patient** : migration vers VM dédiée Windows Server 2022 — élimination des lenteurs
- **Imagerie** : VM stockage haute disponibilité avec PBS + réplication
- **Nomadisme** : VPN WireGuard pour praticiens itinérants
- **Sauvegardes** : règle 3-2-1 — PBS local + Azure Backup hors site, conformité RGPD
- **PRA** : plan de reprise documenté, RTO < 4 h


#### Notre groupe : 
 - OZOUX Guillaume
 - BELMAMOUN Hicham
 - DURANDEAU Emeric
 - MARQUIE Médéric
