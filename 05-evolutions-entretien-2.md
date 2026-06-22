# 05 — Évolutions et pistes pour l'entretien 2

> Ce document est un **guide de préparation** pour l'entretien 2 du MSPR MEDISOL. Il liste les axes d'évolution identifiés lors de la mise en œuvre, les questions techniques et réglementaires à approfondir, les points forts à mettre en avant, et les ressources bibliographiques associées.

---

## 5.1 Évolutions à court terme (0–12 mois)

### 5.1.1 Migration du portail patient vers une solution éprouvée

| Action | Détail | Priorité |
|---|---|---|
| Évaluation éditeur portail patient | Vérifier si l'éditeur propose une offre SaaS hébergée conforme RGPD | Haute |
| Durcissement VM-WEB | WAF applicatif, scan de vulnérabilités mensuel (OpenVAS) | Haute |
| Mise en conformité HDS | Si le portail stocke des données de santé : vérifier exigences hébergement de données de santé (HDS) | Haute |
| Automatisation TLS | Renouvellement automatique Let's Encrypt | Moyenne |

### 5.1.2 Renforcement de la sécurité RGPD

- [ ] **Journalisation complète** des accès au logiciel patient (logs RDS + logs applicatifs éditeur)
- [ ] **DLP (Data Loss Prevention)** : empêcher la copie de données patients vers des supports amovibles (GPO Windows)
- [ ] **Chiffrement de bout en bout** des sauvegardes VEEAM (clé stockée dans coffre-fort hors site)
- [ ] **Audit trimestriel** des accès praticiens (qui a accédé à quels dossiers, depuis où)

### 5.1.3 Optimisation Wi-Fi

- [ ] Passage en **Wi-Fi 6E** (6 GHz) pour les cabinets si les équipements de mesure le supportent → réduction des interférences
- [ ] **Roaming 802.11r** (Fast BSS Transition) entre AP1 et AP2 pour les appareils mobiles praticiens
- [ ] **QoS DSCP** sur le switch : prioriser le trafic RDS/RemoteApp sur le Wi-Fi praticiens

---

## 5.2 Évolutions à moyen terme (12–36 mois)

### 5.2.1 Ouverture d'un cabinet secondaire

Si MEDISOL ouvre un deuxième site :

| Action | Détail |
|---|---|
| Extension cluster Proxmox | 3e nœud sur le site secondaire |
| VPN site-à-site | Tunnel OpenVPN Sophos permanent entre les deux sites |
| Réplication VEEAM inter-sites | VM-PATIENT et VM-IMAGERIE répliquées sur le site secondaire |
| AD Sites and Services | Si Active Directory est déployé à terme |

### 5.2.2 Migration vers un logiciel patient SaaS

Le logiciel patient actuel (client lourd Windows) est vieillissant. Des alternatives SaaS certifiées HDS existent (Doctolib, Maiia, etc.) :

- **Avantage** : plus besoin de VM-PATIENT dédiée + RDS → économie CAPEX/OPEX
- **Inconvénient** : dépendance internet, coût d'abonnement, migration des données historiques
- **À arbitrer** lors de l'entretien 2 : TCO cloud vs on-prem sur 5 ans, délai de migration

### 5.2.3 Imagerie médicale avancée (évolution activité)

Si MEDISOL étend son activité à l'imagerie médicale réglementée (radiologie, scanner) :

- Obligation d'hébergement certifié **HDS** (Hébergeur de Données de Santé)
- Migration de VM-IMAGERIE vers un cloud HDS (OVHcloud Hosted Private Cloud HDS, ou Outscale)
- Intégration DICOM pour les équipements d'imagerie médicale

---

## 5.5 Bibliographie et ressources complémentaires

| Ressource | Référence |
|---|---|
| Documentation officielle Proxmox VE | https://pve.proxmox.com/wiki/Main_Page |
| Proxmox — Corosync QDevice | https://pve.proxmox.com/wiki/Cluster_Manager#_corosync_external_vote_support |
| Sophos Documentation | https://docs.sophos.com/nsg/sophos-utm/utm/9.708/help/en-us/Content/utm/utmAdminGuide/SupportDocumentation.htm |
| CNIL — Guide pratique RGPD pour les professions de santé | https://www.cnil.fr/fr/les-bases-legales/les-conditions-du-traitement-des-donnees-de-sante |
| Certification HDS — Arrêté du 26 février 2018 | https://www.legifrance.gouv.fr/loda/id/JORFTEXT000036637587 |
| ANS — Référentiel HDS (Agence du Numérique en Santé) | https://esante.gouv.fr/produits-services/hds |
| ANSSI — Guide hygiène informatique v2 | ANSSI-GP-078 — https://www.ssi.gouv.fr/guide/guide-dhygiene-informatique/ |
| Code de la santé publique — Art. R. 1112-7 (conservation dossier médical) | https://www.legifrance.gouv.fr/codes/article_lc/LEGIARTI000006912232 |
| Azure Backup — Documentation Microsoft | https://learn.microsoft.com/fr-fr/azure/backup/ |
| Microsoft — Avenant HDS Azure (Hébergeur de Données de Santé) | https://aka.ms/healthdataprotectionaddendum |
| NIST SP 800-34 — Guide PRA | https://csrc.nist.gov/publications/detail/sp/800-34/rev-1/final |
