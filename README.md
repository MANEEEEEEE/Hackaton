# Mini SOC PME — M1Tech Solutions
### Hackathon M1 — Infrastructures Réseaux & Cybersécurité
**Auteur :** Julien Fouquet  
**Date :** 18-19 juin 2026  
**Durée :** 48 heures — Travail individuel

---

## Présentation du projet

Déploiement d'un mini SOC (Security Operations Center) fonctionnel pour la PME fictive **M1Tech Solutions** (80 collaborateurs).

L'infrastructure permet de :
- Héberger un site web institutionnel et une base de données
- Superviser la disponibilité des services en temps réel
- Détecter et centraliser les événements de sécurité via un SIEM
- Analyser les incidents de sécurité

---

## Architecture

```
INTERNET / Attaquants
        │
        ▼
🔥 UFW Firewall (ports 22, 80, 443, 3001, 61208)
        │
        ▼
VM Hôte — Debian GNU/Linux 13 — 192.168.158.132
├── Stack Wazuh SIEM
│   ├── wazuh-manager    (ports 1514, 1515, 55000)
│   ├── wazuh-indexer    (port 9200)
│   └── wazuh-dashboard  (port 443 HTTPS)
│
└── Réseau Docker interne (m1tech-internal-net)
    ├── m1tech-web       nginx:alpine        (port 80)
    ├── m1tech-db        mariadb:10.11       (port 3306 interne)
    ├── m1tech-supervision uptime-kuma:1     (port 3001)
    └── m1tech-security-monitor glances      (port 61208)
```

---

## Stack technique

| Composant | Technologie | Version | Rôle |
|-----------|-------------|---------|------|
| OS hôte | Debian GNU/Linux | 13 (trixie) | Système hôte |
| Conteneurs | Docker + Compose | latest | Orchestration |
| Site web | Nginx | Alpine | Intranet institutionnel |
| Base de données | MariaDB | 10.11 | Stockage données métiers |
| Supervision | Uptime Kuma | 1.x | Disponibilité des services |
| Métriques | Glances | latest-full | Monitoring système |
| SIEM | Wazuh | 4.12.0 | Détection + corrélation |
| IDS/FIM | Wazuh Agent | 4.12.0 | Surveillance hôte |
| Pare-feu | UFW | 0.36 | Filtrage réseau |
| Anti-brute | Fail2Ban | 1.1.0 | Protection SSH |

---

## Prérequis

- Machine Linux (Debian 11+ ou Ubuntu 22.04+)
- Minimum 8 Go RAM, 40 Go disque, 4 vCPU
- Docker + Docker Compose installés
- Accès root ou sudo

---

## Installation et reproduction

### 1. Cloner le dépôt

```bash
git clone https://github.com/<votre-username>/m1tech-soc.git
cd m1tech-soc
```

### 2. Déployer le mini-SOC (4 conteneurs)

```bash
cd mini-soc/
docker compose up -d
docker ps
```

Vérifier l'accès :
- Site web : http://\<IP\>
- Uptime Kuma : http://\<IP\>:3001
- Glances : http://\<IP\>:61208

### 3. Déployer Wazuh SIEM

```bash
# Paramètre obligatoire pour OpenSearch
sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" >> /etc/sysctl.conf

# Cloner le dépôt officiel Wazuh
cd /opt
git clone https://github.com/wazuh/wazuh-docker.git -b v4.12.0 --depth 1
cd /opt/wazuh-docker/single-node

# Générer les certificats SSL
docker compose -f generate-indexer-certs.yml run --rm generator

# Lancer la stack
docker compose up -d
```

Accès dashboard Wazuh : https://\<IP\>  
Login : `admin` / `SecretPassword`

### 4. Installer l'agent Wazuh sur la VM hôte

```bash
# Ajouter le dépôt Wazuh
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" > /etc/apt/sources.list.d/wazuh.list
apt update

# Installer la version 4.12.0 (doit correspondre au manager)
WAZUH_MANAGER="127.0.0.1" WAZUH_AGENT_NAME="M1Tech-Server" apt install wazuh-agent=4.12.0-1 -y

# Démarrer l'agent
systemctl enable wazuh-agent
systemctl start wazuh-agent
```

### 5. Configurer le pare-feu UFW

```bash
export PATH=$PATH:/usr/sbin
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 3001/tcp
ufw allow 61208/tcp
ufw enable
ufw status numbered
```

### 6. Durcir SSH

```bash
sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/^#MaxAuthTries.*/MaxAuthTries 3/' /etc/ssh/sshd_config
systemctl restart ssh
```

### 7. Installer Fail2Ban

```bash
apt install fail2ban -y
cat > /etc/fail2ban/jail.d/ssh.conf << 'EOF'
[sshd]
enabled = true
maxretry = 3
bantime = 3600
findtime = 600
EOF
systemctl enable fail2ban
systemctl start fail2ban
```

---

## Structure du dépôt

```
m1tech-soc/
├── README.md
├── mini-soc/
│   ├── docker-compose.yml          # Stack mini-SOC (Nginx, MariaDB, Uptime Kuma, Glances)
│   └── nginx/
│       └── index.html              # Page HTML du site institutionnel
├── wazuh/
│   └── docker-compose-wazuh.yml    # Stack Wazuh SIEM (Manager, Indexer, Dashboard)
├── commandes.txt                   # Journal complet des commandes utilisées
└── schemas/
    └── schema_reseau_M1Tech.drawio # Schéma réseau et logique de l'infrastructure
```

---

## Événements de sécurité démontrés

| # | Événement | Outil de détection | Résultat |
|---|-----------|-------------------|----------|
| 1 | Brute-force SSH (fakeuser) | Wazuh + journalctl | 11 auth failures — MITRE T1110 |
| 2 | Scan de reconnaissance Nmap | UFW + logs SSH | Ports 80/443/3001 filtrés |
| 3 | Modification /etc/hosts | Wazuh FIM | Rule 550 Level 7 — checksum changed |

---

## Résultats obtenus

- **906 alertes** détectées par Wazuh en quelques heures
- **Détection de panne** en moins de 20 secondes (Uptime Kuma)
- **MITRE ATT&CK** : Password Guessing, SSH, Brute Force, Create Account
- **FIM** : modification de fichier système détectée en temps réel

---

## Axes d'amélioration

- [ ] HTTPS avec Certbot sur Nginx (reverse proxy)
- [ ] VPN WireGuard pour accès distant sécurisé
- [ ] Authentification MFA (TOTP)
- [ ] Wazuh Active Response (blocage automatique des IP)
- [ ] Sauvegarde automatisée avec restic
- [ ] Scans de vulnérabilités avec Trivy/OpenVAS

---

## Licence

Projet réalisé dans le cadre du Hackathon M1 — Infrastructures Réseaux & Cybersécurité.
