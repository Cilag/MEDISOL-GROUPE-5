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

Configuration via l'interface Sophos → Firewall → Rules :

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

VM Linux (Ubuntu) hébergeant la stack de supervision **Prometheus + Alertmanager + Grafana**,
avec les exporters **Node**, **Windows**, **Blackbox** (sondes HTTP) et **SNMP** (firewall Sophos).
Installée via snap (`/var/snap/prometheus/current/`).

```
Ressources : 2 vCPU, 4 GB RAM, 40 GB disque
IP         : 10.30.0.x (VLAN 30 INFRA)
Services   : prometheus (9090), alertmanager (9093), grafana (3000),
             node_exporter (9100), blackbox_exporter (9115), snmp_exporter (9116)
```

**Cibles supervisées :**

| Cible | Adresse | Exporter | Rôle |
|---|---|---|---|
| vm-mon | localhost:9100 | node | Supervision |
| vm-web | 10.30.0.5:9100 + Blackbox HTTP | node / blackbox | Portail patient |
| ad-01 | 10.30.0.1:9182 | windows | Active Directory (principal) |
| ad-02 | 10.30.0.2:9182 | windows | Active Directory (réplication) |
| fic-01 | 10.30.0.3:9182 | windows | Serveur de fichiers |
| veeam-01 | 10.30.0.4:9182 | windows | Sauvegarde |
| sophos-xg | 10.30.0.254 | snmp (if_mib) | Pare-feu |
| postes | `postes.json` (file_sd) | windows | Postes clients (découverte dynamique) |

#### `/var/snap/prometheus/current/prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    organisation: 'medisol'
    env: 'production'

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']

rule_files:
  - "/var/snap/prometheus/current/rules/*.yml"

scrape_configs:

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
        labels:
          hostname: 'vm-mon'
          role: 'monitoring'

  - job_name: 'vm-mon-node'
    static_configs:
      - targets: ['localhost:9100']
        labels:
          hostname: 'vm-mon'
          role: 'monitoring'

  - job_name: 'vm-web-node'
    static_configs:
      - targets: ['10.30.0.5:9100']
        labels:
          hostname: 'vm-web'
          role: 'portail-patient'

  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - 'http://10.30.0.5'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: 'localhost:9115'
      - source_labels: [__param_target]
        target_label: instance

  - job_name: 'ad-dc1'
    static_configs:
      - targets: ['10.30.0.1:9182']
        labels:
          hostname: 'ad-01'
          role: 'active-directory'
          dc: 'principal'

  - job_name: 'ad-dc2'
    static_configs:
      - targets: ['10.30.0.2:9182']
        labels:
          hostname: 'ad-02'
          role: 'active-directory'
          dc: 'replication'

  - job_name: 'fic-node'
    static_configs:
      - targets: ['10.30.0.3:9182']
        labels:
          hostname: 'fic-01'
          role: 'fileserver'

  - job_name: 'veeam-node'
    static_configs:
      - targets: ['10.30.0.4:9182']
        labels:
          hostname: 'veeam-01'
          role: 'backup'

  - job_name: 'sophos'
    metrics_path: /snmp
    params:
      module: [if_mib]
      auth: [medisol_v2]
    static_configs:
      - targets: ['10.30.0.254']
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: 'localhost:9116'
      - source_labels: [__param_target]
        target_label: instance
        replacement: 'sophos-xg'

  - job_name: 'postes'
    file_sd_configs:
      - files: ['/var/snap/prometheus/current/targets/postes.json']
        refresh_interval: 1m
```

#### `/var/snap/prometheus/current/rules/alertes.yml`

Règles d'alerte regroupées par domaine (disponibilité, ressources Linux/Windows, Proxmox,
Active Directory, Veeam, portail patient, Sophos).

```yaml
groups:

  - name: medisol.disponibilite
    rules:
      - alert: HostDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Host inaccessible — {{ $labels.instance }}"
          description: "La cible {{ $labels.instance }} ne répond plus depuis 1 minute."

  - name: medisol.ressources-linux
    rules:
      - alert: CPUEleve
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 75
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "CPU eleve — {{ $labels.instance }}"
          description: "CPU a {{ $value | printf \"%.0f\" }}% depuis 10 min."

      - alert: CPUCritique
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "CPU critique — {{ $labels.instance }}"
          description: "CPU a {{ $value | printf \"%.0f\" }}% depuis 5 min."

      - alert: RAMCritique
        expr: (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 < 10
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "RAM critique — {{ $labels.instance }}"
          description: "Memoire disponible {{ $value | printf \"%.1f\" }}%. Risque OOM."

      - alert: DisqueEleve
        expr: (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes) * 100 > 80
        for: 15m
        labels: { severity: warning }
        annotations:
          summary: "Disque > 80% — {{ $labels.instance }} {{ $labels.mountpoint }}"
          description: "Partition {{ $labels.mountpoint }} a {{ $value | printf \"%.0f\" }}%."

      - alert: DisqueCritique
        expr: (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes) * 100 > 95
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "Disque > 95% — {{ $labels.instance }} {{ $labels.mountpoint }}"
          description: "Espace disque quasi sature sur {{ $labels.instance }}."

  - name: medisol.ressources-windows
    rules:
      - alert: WindowsCPUEleve
        expr: 100 - (avg by (instance) (rate(windows_cpu_time_total{mode="idle"}[5m])) * 100) > 80
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "CPU Windows eleve — {{ $labels.instance }}"
          description: "CPU Windows a {{ $value | printf \"%.0f\" }}% depuis 10 min."

      - alert: WindowsRAMCritique
        expr: (windows_os_physical_memory_free_bytes / windows_cs_physical_memory_bytes) * 100 < 10
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "RAM Windows critique — {{ $labels.instance }}"
          description: "Moins de 10% de RAM libre sur {{ $labels.instance }}."

      - alert: WindowsDisqueCritique
        expr: (1 - windows_logical_disk_free_bytes / windows_logical_disk_size_bytes) * 100 > 90
        for: 10m
        labels: { severity: critical }
        annotations:
          summary: "Disque Windows > 90% — {{ $labels.instance }} {{ $labels.volume }}"
          description: "Volume {{ $labels.volume }} a {{ $value | printf \"%.0f\" }}%."

      - alert: ServiceWindowsArrete
        expr: windows_service_status{status="stopped",start_mode="auto"} == 1
        for: 2m
        labels: { severity: critical }
        annotations:
          summary: "Service Windows arrete — {{ $labels.name }} sur {{ $labels.instance }}"
          description: "Le service {{ $labels.name }} en demarrage auto est arrete."

  - name: medisol.proxmox
    rules:
      - alert: ProxmoxVMDown
        expr: pve_up{type="qemu"} == 0
        for: 2m
        labels: { severity: critical }
        annotations:
          summary: "VM Proxmox arretee — {{ $labels.name }}"
          description: "La VM {{ $labels.name }} n'est plus en etat running."

  - name: medisol.active-directory
    rules:
      - alert: ADReplicationEchec
        expr: ad_replication_failures_total > 0
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "Echec replication AD — {{ $labels.instance }}"
          description: "Erreurs de replication AD detectees sur {{ $labels.instance }}."

      - alert: ADComptesBloques
        expr: ad_locked_accounts_count > 5
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "{{ $value }} comptes AD verrouilles"
          description: "Possible bruteforce ou probleme de synchro mot de passe."

      - alert: ADDCIndisponible
        expr: up{job=~"ad-dc1|ad-dc2"} == 0
        for: 1m
        labels: { severity: critical }
        annotations:
          summary: "DC inaccessible — {{ $labels.instance }}"
          description: "Le controleur de domaine {{ $labels.instance }} ne repond plus."

  - name: medisol.veeam
    rules:
      - alert: VeeamJobEchec
        expr: veeam_job_result_value > 0
        for: 0m
        labels: { severity: critical }
        annotations:
          summary: "Backup Veeam en echec — {{ $labels.job_name }}"
          description: "Le job {{ $labels.job_name }} a echoue."

      - alert: VeeamPasDeBackup
        expr: (time() - veeam_job_last_run_time_seconds) / 3600 > 26
        for: 30m
        labels: { severity: warning }
        annotations:
          summary: "Pas de backup depuis {{ $value | printf \"%.0f\" }}h"
          description: "Le job {{ $labels.job_name }} n'a pas tourne depuis plus de 26h."

  - name: medisol.portail
    rules:
      - alert: PortailDown
        expr: probe_success{job="blackbox-http"} == 0
        for: 2m
        labels: { severity: critical }
        annotations:
          summary: "Portail Patient inaccessible — {{ $labels.instance }}"
          description: "La probe HTTP sur {{ $labels.instance }} a echoue."

      - alert: PortailLent
        expr: probe_duration_seconds{job="blackbox-http"} > 3
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "Portail Patient lent — {{ $value | printf \"%.1f\" }}s"
          description: "Le portail repond en {{ $value | printf \"%.1f\" }}s (seuil 3s)."

  - name: medisol.sophos
    rules:
      - alert: SophosChargeElevee
        expr: hrProcessorLoad > 70
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "CPU Sophos > 70%"
          description: "Charge CPU firewall elevee. Verifier regles IPS ou pic trafic."
```

#### `/etc/blackbox_exporter/config.yml` — sonde HTTP du portail

```yaml
modules:
  http_2xx:
    prober: http
    timeout: 10s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      method: GET
      follow_redirects: true
      fail_if_not_ssl: false
```

#### Grafana & SNMP exporter

- **`/etc/grafana/grafana.ini`** : configuration laissée **par défaut** (toutes les directives
  fonctionnelles sont commentées) — Grafana sert uniquement de visualisation, branché sur la
  datasource Prometheus `localhost:9090`. Seul le module *recording rules* est actif.
- **`/etc/snmp_exporter/snmp.yml`** : fichier généré par le *generator* SNMP (blocs `auths:` et
  `modules:`), volontairement non recopié ici (≈1,2 Mo d'OID). L'authentification utilisée est
  `medisol_v2` (SNMP v2c) et le module `if_mib` pour la collecte des interfaces du Sophos.

### VM-WEB — Portail patient (DMZ)

Serveur **Nginx** (Debian/Ubuntu) hébergeant le portail patient, placé en DMZ et exposé
uniquement via le reverse-proxy/pare-feu Sophos.

```
Ressources : 2 vCPU, 2 GB RAM, 20 GB disque
Réseau     : VLAN dédié DMZ — accès entrant HTTP/HTTPS (80/443) depuis le WAN via Sophos
Service    : nginx (www-data)
Racine web : /var/www/html
```

#### `/etc/nginx/sites-available/default` — virtual host

```nginx
# Default server configuration
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        # SSL configuration (à activer en production avec un certificat valide)
        # listen 443 ssl default_server;
        # listen [::]:443 ssl default_server;
        # include snippets/snakeoil.conf;   # NE PAS utiliser en prod

        root /var/www/html;
        index index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                # Sert le fichier, puis le répertoire, sinon 404
                try_files $uri $uri/ =404;
        }

        # FastCGI / PHP (désactivé — portail statique)
        #location ~ \.php$ {
        #       include snippets/fastcgi-php.conf;
        #       fastcgi_pass unix:/run/php/php7.4-fpm.sock;
        #}

        # Interdire l'accès aux fichiers .htaccess
        #location ~ /\.ht {
        #       deny all;
        #}
}
```

#### `/etc/nginx/nginx.conf` — configuration globale

```nginx
user www-data;
worker_processes auto;
worker_cpu_affinity auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
        # multi_accept on;
}

http {
        ##
        # Basic Settings
        ##
        sendfile on;
        tcp_nopush on;
        types_hash_max_size 2048;
        server_tokens build;            # masquer la version de Nginx (durcissement)

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ##
        # SSL Settings
        ##
        ssl_protocols TLSv1.2 TLSv1.3;  # SSLv3/TLS 1.0/1.1 désactivés (POODLE)
        ssl_prefer_server_ciphers off;

        ##
        # Logging Settings
        ##
        access_log /var/log/nginx/access.log;

        ##
        # Gzip Settings
        ##
        gzip on;

        ##
        # Virtual Host Configs
        ##
        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}
```

> **Durcissement appliqué** : `server_tokens` masqué, protocoles limités à TLS 1.2/1.3,
> PHP désactivé (portail statique). En production, activer le bloc SSL avec un certificat
> valide (Let's Encrypt) et non le certificat auto-signé `snakeoil`.

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
