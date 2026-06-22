# 01 — Contexte et analyse du besoin

## 1.1 Présentation du client

**MEDISOL** est un centre de consultations et d'imagerie légère spécialisé en bien-être et médecine douce, situé en France, comptant **32 personnes** réparties entre :
- Le personnel d'**accueil** (postes partagés, gestion des rendez-vous)
- Les **praticiens** (ostéopathes, naturopathes, kinésithérapeutes, etc.) — dont une partie est **nomade** (1 jour / semaine hors site)
- L'équipe **administrative** (facturation, mutuelles, télétransmission carte Vitale)

Le centre exploite également des **équipements de mesure connectés** en cabinet (tensiomètres, ECG léger, appareils d'imagerie légère) et dispose d'un système de **contrôle d'accès et de vidéosurveillance**.

L'équipe technique est réduite et externalisée : elle est **hébergée dans un local proche** du site principal (espace externalisé à proximité), ce qui confirme l'absence de ressources IT internes permanentes sur site.

---

## 1.2 Situation technique actuelle

### Infrastructure existante

| Composant | Description |
|---|---|
| Serveur physique | 1 serveur **Windows Server** hébergeant le stockage imagerie |
| Logiciel patient | Client lourd Windows (licence propriétaire, éditeur tiers) — accès sur postes d'accueil |
| Réseau | Réseau **plat**, aucune séparation VLAN |
| Wi-Fi | Un seul SSID partagé entre patients en salle d'attente et personnel |
| Messagerie / Bureautique | **Microsoft 365** |
| Prise de RDV | Portail en ligne existant |
| Portail patient | En cours de développement |
| Sauvegarde | Aucune politique formelle documentée |
| Accès nomades | Non sécurisé (solutions ad hoc) |
| Volume de données | **5 TB actuellement**, croissance de **10 GB/mois** — données de santé soumises au RGPD |

### Points de défaillance critiques identifiés

1. **Réseau non segmenté** : les patients en salle d'attente partagent le même segment réseau que les postes de travail du personnel — risque de fuite de données patients.
2. **Wi-Fi saturé** : la bande passante unique est consommée par les smartphones patients, dégradant la qualité de service pour les applicatifs métier.
3. **SPOF stockage imagerie** : un seul serveur physique héberge les données d'imagerie sans réplication ni haute disponibilité.
4. **Logiciel patient lent** : aux heures de pointe, le client lourd Windows accède à des ressources sur un serveur non optimisé, causant des ralentissements visibles.
5. **Données patients exposées** : absence de contrôle réseau entre la zone invitée et les serveurs métier — non conforme RGPD.
6. **Aucun PRA documenté** : pas de plan de reprise après incident, alors que la direction exige une disponibilité « zéro panne ».
7. **Accès praticiens nomades non sécurisé** : les praticiens intervenant hors site n'ont pas d'accès sécurisé aux dossiers patients.

### Situation réseau actuelle

```
[Internet]
    │
[Box FAI]
    │
[Switch non manageable]
    ├── Postes accueil (logiciel patient)
    ├── Postes administration (facturation, M365)
    ├── Serveur Windows Server (imagerie)
    ├── Wi-Fi patients (salle d'attente)    ← MÊME RÉSEAU
    ├── Wi-Fi praticiens
    └── Équipements connectés (mesures, caméras, contrôle accès)
```

---

## 1.3 Objectifs exprimés par la direction

### Priorité 1 — Disponibilité totale
> *"Zéro panne — un arrêt du système bloque toute l'activité clinique."*

- **RTO cible** : < 2 heures en cas de panne d'un équipement
- **RPO cible** : < 24 heures de perte de données maximale
- Tolérance aux pannes sans interruption visible des praticiens et de l'accueil

### Priorité 2 — Confidentialité des données patients
- Conformité **RGPD** : isolation stricte des données patients
- Séparation réseau entre zone invités, zone praticiens/accueil et zone administration
- Aucune donnée patient accessible depuis le réseau Wi-Fi patients

### Priorité 3 — Résoudre la saturation Wi-Fi et les lenteurs applicatives
- Déploiement d'une infrastructure Wi-Fi managée avec **SSIDs distincts par VLAN**
- Amélioration des performances du **logiciel patient** via virtualisation dédiée

### Priorité 4 — Accès nomades sécurisés
- Les praticiens itinérants (1 jour / semaine hors site) doivent accéder aux dossiers patients de façon sécurisée
- **VPN** avec authentification forte (MFA)

### Priorité 5 — Portail patient et prise de RDV en ligne
- Le portail patient en développement doit être hébergé dans un environnement sécurisé et isolé
- Haute disponibilité de la prise de RDV en ligne (impact direct sur l'activité)

---

## 1.4 Contraintes techniques

| Contrainte | Détail |
|---|---|
| **Logiciel patient tiers** | Client lourd Windows — migration à coordonner avec l'éditeur ; pas de remplacement immédiat |
| **Données imagerie existantes** | Migration des données depuis le Windows Server actuel sans perte |
| **Équipements connectés** | Tensiomètres, ECG, caméras, contrôle d'accès — nécessitent un VLAN dédié IoT |
| **Contrainte RGPD** | Données de santé → chiffrement au repos et en transit, journalisation des accès |
| **Continuité pendant migration** | Pas d'arrêt acceptable pendant les heures de consultation (L-S 8h-20h) |
| **Compétences IT faibles** | Pas d'IT interne — la solution doit être administrable par un prestataire externe |
| **Surface utile limitée** | Petit local technique — 2 serveurs rack 1U/2U maximum |
| **Volume de données RGPD** | 5 TB actuels + 10 GB/mois de croissance — stratégie de stockage et archivage RGPD à planifier |

---

## 1.5 Contraintes budgétaires

Le client a confirmé la disponibilité d'un **budget large** pour ce projet. Il n'existe pas de contrainte financière particulière — la proposition peut intégrer des solutions robustes et redondantes sans compromettre l'acceptation.

Les points clés validés par le client :

- **Matériel** : 2 serveurs physiques compacts (format 1U/2U) — le dimensionnement peut être revu à la hausse si la redondance ou les performances l'exigent
- **Licences** : préférence open-source ; les licences Microsoft 365 sont déjà en place
- **OPEX** : coût mensuel maîtrisé, pas de full-cloud imprévisible
- **Prestataire unique** : maintenance externalisée à un seul prestataire

### Estimation budgétaire prévisionnelle

Les chiffres ci-dessous constituent une base de référence. Ils peuvent être dépassés si des choix techniques justifiés apportent une meilleure redondance ou de meilleures performances.

| Poste | Estimation (HT) |
|---|---|
| 2x serveurs physiques (Dell R350 ou équivalent) | 8 000 – 14 000 € |
| 2x Switch manageable 24 ports avec PoE (pour AP Wi-Fi) | 1 200 – 2 000 € |
| 2x Access Points Wi-Fi 6 managés | 600 – 1 000 € |
| Licences Windows Server 2022 (2x Standard, 32 CAL) | 3 000 – 4 500 € |
| Prestations migration + configuration | 4 000 – 7 000 € |
| **Total CAPEX estimé** | **~17 000 – 29 000 €** |

> Ces estimations sont indicatives. Le client ayant confirmé un budget large, des options complémentaires (stockage redondant supplémentaire, licences entreprise, matériel haute disponibilité) peuvent être proposées et chiffrées séparément.

---

## 1.6 Synthèse du besoin

```
Situation actuelle    → Problème             → Besoin
──────────────────────────────────────────────────────────────────────
Réseau plat          → Données patients      → VLANs + pare-feu OPNsense
                       exposées              
Wi-Fi unique         → Saturation + risque   → SSIDs par VLAN + QoS
1 serveur imagerie   → SPOF données          → 2 nœuds Proxmox + réplication
Logiciel patient     → Lenteurs aux pointes  → VM dédiée WS 2022
  sur poste physique                         
Accès nomades        → Non sécurisé          → OpenVPN + MFA
  non sécurisés
Pas de PRA           → 0 plan de reprise     → PBS + Azure Backup + PRA écrit     # A corriger PBS
Portail patient      → Hébergement précaire  → VM DMZ dédiée
  en projet
5 TB + 10 GB/mois   → Croissance données    → Stockage scalable + archivage RGPD
```
