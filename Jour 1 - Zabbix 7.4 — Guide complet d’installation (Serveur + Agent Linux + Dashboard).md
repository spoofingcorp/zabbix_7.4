# 🧭 Atelier Complet Zabbix 7.4 — Ubuntu 24.04  
**Installation Serveur | Installation Agent Linux | Ajout d’Hôte | Templates | Dashboard | Vérifications | Dépannage**

---

---

# 🧭 Table des matières

* [1. Installation du serveur Zabbix 7.4 (Ubuntu 24.04)](#1-installation-du-serveur-zabbix-74-ubuntu-2404)

  * [1.1 Script complet (auto-install)](#11-script-complet-auto-install)
  * [1.2 Lancer l’installation](#12-lancer-linstallation)
  * [1.3 Configuration Web initiale](#13-configuration-web-initiale)
  * [1.4 Correction ICMP (fping)](#14-correction-icmp-fping)
* [2. Installation agent Zabbix 7.4 pour Linux (Ubuntu 24.04)](#2-installation-agent-zabbix-74-pour-linux-ubuntu-2404)

  * [2.1 Installation du dépôt (source officielle)](#21-installation-du-dépôt-source-officielle)
  * [2.2 Installation de l’agent](#22-installation-de-lagent)
  * [2.3 Configuration de l’agent](#23-configuration-de-lagent)
  * [2.4 Tests depuis le serveur Zabbix](#24-tests-depuis-le-serveur-zabbix)
* [3. Ajouter un hôte Linux dans Zabbix 7.4](#3-ajouter-un-hôte-linux-dans-zabbix-74)

  * [3.1 Paramètres essentiels](#31-paramètres-essentiels)
  * [3.2 Templates Linux recommandés (Zabbix 7.4)](#32-templates-linux-recommandés-zabbix-74)
* [4. Création d’un Dashboard](#4-création-dun-dashboard)
* [5. Dépannage](#5-dépannage)

---

# 1. **Installation du serveur Zabbix 7.4 (Ubuntu 24.04)**

## 1.1 **Script complet (auto-install)**

Créer le fichier :

```bash
nano /usr/local/sbin/install-zabbix74-full.sh
```

Coller :

```bash
#!/bin/bash
set -euo pipefail

####################################################################
#  INSTALLATION ZABBIX 7.4 + POSTGRESQL 16 + NGINX + AGENT2
#  Ubuntu Server 24.04 — Full Automatic Installer (NGINX FIXED + LOCALES)
####################################################################

DB_PASS="MotDePasse!"
TZ="Europe/Paris"

echo "=== 🧭 Mise à jour système ==="
apt update && apt upgrade -y
timedatectl set-timezone "$TZ"

############################################################
# INSTALLATION POSTGRESQL 16
############################################################

echo "=== 🗄️ Installation PostgreSQL ==="
apt install -y postgresql postgresql-contrib

echo "=== 🔍 PostgreSQL version ==="
psql --version

PG_VERSION=$(psql -V | awk '{print $3}' | cut -d. -f1)
PG_CONF_DIR="/etc/postgresql/${PG_VERSION}/main"
PG_OPT_CONF="${PG_CONF_DIR}/conf.d/zabbix-optim.conf"

############################################################
# CRÉATION BDD ZABBIX
############################################################

echo "=== 🗃️ Création base et utilisateur Zabbix ==="
sudo -u postgres psql <<EOF
CREATE USER zabbix WITH PASSWORD '${DB_PASS}';
CREATE DATABASE zabbix OWNER zabbix ENCODING 'UTF8';
EOF

############################################################
# INSTALLATION ZABBIX 7.4
############################################################

echo "=== 📦 Installation dépôt Zabbix 7.4 ==="
cd /tmp
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.4+ubuntu24.04_all.deb
apt update

echo "=== 📦 Installation paquets Zabbix ==="
apt install -y \
    zabbix-server-pgsql \
    zabbix-frontend-php \
    php8.3-pgsql \
    zabbix-nginx-conf \
    zabbix-sql-scripts \
    zabbix-agent2

############################################################
# IMPORT DU SCHEMA SQL
############################################################

echo "=== 📥 Import du schéma SQL ==="
zcat /usr/share/zabbix/sql-scripts/postgresql/server.sql.gz \
    | sudo -u zabbix psql zabbix

############################################################
# CONFIGURATION ZABBIX SERVER
############################################################

echo "=== ⚙️ Configuration Zabbix Server ==="
ZBX_CONF="/etc/zabbix/zabbix_server.conf"

sed -i "s/# DBPassword=/DBPassword=${DB_PASS}/" "$ZBX_CONF"

cat >> "$ZBX_CONF" <<EOF

### Optimisations ###
StartPollers=20
StartPollersUnreachable=6
StartTrappers=10
StartDiscoverers=4
CacheSize=512M
HistoryCacheSize=256M
TrendCacheSize=64M
ValueCacheSize=256M
EOF

############################################################
# OPTIMISATION POSTGRESQL
############################################################

echo "=== ⚡ Optimisation automatique PostgreSQL ==="

mkdir -p "${PG_CONF_DIR}/conf.d"

TOTAL_RAM_KB=$(grep MemTotal /proc/meminfo | awk '{print $2}')
TOTAL_RAM_MB=$(( TOTAL_RAM_KB / 1024 ))
CPU_COUNT=$(nproc)

SHARED_BUFFERS_MB=$(( TOTAL_RAM_MB / 4 ))
EFFECTIVE_CACHE_MB=$(( TOTAL_RAM_MB * 3 / 4 ))
MAINT_WORK_MB=$(( TOTAL_RAM_MB / 8 ))

if [ "$SHARED_BUFFERS_MB" -lt 512 ]; then SHARED_BUFFERS_MB=512; fi
if [ "$MAINT_WORK_MB" -lt 128 ]; then MAINT_WORK_MB=128; fi

cat <<EOF > "$PG_OPT_CONF"
# ===================================================================
#  Optimisations PostgreSQL pour Zabbix 7.4
# ===================================================================

shared_buffers = ${SHARED_BUFFERS_MB}MB
effective_cache_size = ${EFFECTIVE_CACHE_MB}MB
work_mem = 32MB
maintenance_work_mem = ${MAINT_WORK_MB}MB

checkpoint_completion_target = 0.9
max_wal_size = 4GB
min_wal_size = 1GB

autovacuum = on
autovacuum_vacuum_scale_factor = 0.05
autovacuum_analyze_scale_factor = 0.05
autovacuum_max_workers = 5
autovacuum_naptime = 20s

max_connections = 200
superuser_reserved_connections = 3

wal_buffers = 16MB
wal_writer_delay = 200ms

max_worker_processes = ${CPU_COUNT}
max_parallel_workers = ${CPU_COUNT}
max_parallel_workers_per_gather = 2
EOF

systemctl restart postgresql

############################################################
# CONFIGURATION NGINX + PHP + LOCALES
############################################################

apt install -y locales
locale-gen en_US en_US.UTF-8 fr_FR fr_FR.UTF-8
update-locale

sed -i "s/# listen 80;/listen 80;/" /etc/zabbix/nginx.conf
sed -i "s/# server_name .*/server_name _;/" /etc/zabbix/nginx.conf

if [ -f /etc/nginx/sites-enabled/default ]; then
    rm /etc/nginx/sites-enabled/default
fi

ln -sf /etc/zabbix/nginx.conf /etc/nginx/sites-enabled/zabbix.conf

nginx -t
systemctl restart nginx

echo "php_value[date.timezone] = ${TZ}" >> /etc/php/8.3/fpm/pool.d/zabbix.conf
systemctl restart php8.3-fpm

############################################################
# DÉMARRAGE SERVICES
############################################################

systemctl restart zabbix-server zabbix-agent2 nginx php8.3-fpm
systemctl enable zabbix-server zabbix-agent2 nginx php8.3-fpm

############################################################
# FIN
############################################################

IP=$(hostname -I | awk '{print $1}')

echo ""
echo "==============================================================="
echo " ✔️ INSTALLATION ZABBIX 7.4 TERMINÉE"
echo " 🌐 Interface Web : http://$IP/"
echo " 🔑 Identifiants par défaut : Admin / zabbix"
echo "==============================================================="
```

---

## 1.2 Lancer l’installation

```bash
chmod +x /usr/local/sbin/install-zabbix74-full.sh
sudo bash /usr/local/sbin/install-zabbix74-full.sh
```

---

## 1.3 **Configuration Web initiale**

Une fois l’interface web accessible via :

```
http://ADRESSE_IP_DU_SERVEUR/
```

Renseigner les champs suivants :

| Champ        | Valeur        |
| ------------ | -----------   |
| Type         | PostgreSQL    |
| Hôte         | localhost     |
| Port         | 0             |
| Base         | zabbix        |
| Schéma       |(laisser vide) |
| Utilisateur  | zabbix        |
| Mot de passe | MotDePasse!   |
| TLS          | ❌ désactivé  |

Terminer l’assistant de configuration, puis connecter-vous :

```
Admin / zabbix
```

---

## 1.4 **Correction ICMP (fping)**

Zabbix nécessite `fping` dans `/usr/sbin` pour les tests de disponibilité ICMP.

```bash
apt install fping
cp /usr/bin/fping /usr/sbin
chmod u+s /usr/sbin/fping
```

Zabbix nécessite `zabbix-get` pour tester la communication entre le serveur Zabbix et un agent, mais uniquement depuis le serveur Zabbix.

```bash
apt install zabbix-get
```

---

## 1.5 **Activer les scripts globaux**

```bash
nano /etc/zabbix/zabbix_server.conf
```

Modifier :

```
EnableGlobalScripts=1
```

Redémarrer :

```bash
systemctl restart zabbix-server
```

---

# 2. **Installation agent Zabbix 7.4 pour Linux (Ubuntu 24.04 et Debian 13)**

## 2.1 **Installation du dépôt (source officielle)**

⚠️ ATTENTION SELECTIONNER LA VERSION DE L'AGENT EN FONCTION DE VOTRE OS 😸

Source pour Ubuntu 24.04 :
[https://www.zabbix.com/download?zabbix=7.2&os_distribution=ubuntu&os_version=24.04&components=agent&db=&ws=](https://www.zabbix.com/download?zabbix=7.2&os_distribution=ubuntu&os_version=24.04&components=agent&db=&ws=)

Source pour Debian 13 :
[https://www.zabbix.com/download?zabbix=7.4&os_distribution=debian&os_version=13&components=agent&db=&ws=](https://www.zabbix.com/download?zabbix=7.4&os_distribution=debian&os_version=13&components=agent&db=&ws=)

```bash
# AGENT OU UBUNTU 24.04
wget https://repo.zabbix.com/zabbix/7.2/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.2+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.2+ubuntu24.04_all.deb
apt update
```

```bash
# AGENT OU DEBIAN 12
wget https://repo.zabbix.com/zabbix/7.4/release/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.4+debian13_all.deb
dpkg -i zabbix-release_latest_7.4+debian13_all.deb
apt update
```

---

## 2.2 **Installation de l’agent**

```bash
apt install zabbix-agent -y
```

---

## 2.3 **Configuration de l’agent**

Éditer :

```bash
nano /etc/zabbix/zabbix_agentd.conf
```

Modifier les paramètres essentiels :

### 🔹 Serveur autorisé à interroger l’agent (passif)

```
Server=192.168.X.X  # Renseigner l'IP de votre serveur Zabbix
```

### 🔹 Serveur recevant les données de l’agent (actif)

```
ServerActive=192.168.X.X  # Renseigner l'IP de votre serveur Zabbix
```

### 🔹 Nom de l’hôte (DOIT correspondre dans Zabbix)

```
Hostname=linux-web01
```

Redémarrer l’agent :

```bash
systemctl restart zabbix-agent
systemctl enable zabbix-agent
```

---

## 2.4 **Tests depuis le serveur Zabbix**

Depuis le serveur :

```bash
zabbix_get -s 192.168.x.x -k agent.ping   # Changer par l'IP de votre VM Debian GUI ou Ubuntu GUI qui dipose de l'agent afin de tester sa réponse depuis le serveur Zabbix
```

Résultat attendu :

```
1
```

Cela confirme que :

✔ L’agent répond
✔ Le port 10050 est ouvert
✔ Le hostname est correct
✔ Le serveur Zabbix communique avec l’agent

---

# 3. **Ajouter un hôte Linux dans Zabbix 7.4**

Menu :

```
Data collection → Hosts → Create host
```

---

## 3.1 **Paramètres essentiels**

### 🔹 Host

* **Hostname :** `linux-web01` (identique au fichier agent)
* **Groups :** `Linux servers`

### 🔹 Interface

```
Type : Agent
IP   : 192.168.20.75
Port : 10050
```

---

## 3.2 **Templates Linux recommandés (Zabbix 7.4)**

Depuis Zabbix 6.2+, les templates Linux sont **fusionnés en un seul template principal** :

### ✔ **Linux by Zabbix agent**

Ce template inclut automatiquement :

* CPU monitoring
* Memory monitoring
* Filesystem discovery
* Network interface discovery
* Uptime
* Load average
* Process monitoring
* Triggers CPU / RAM / Disk / Network

### Pourquoi je ne vois plus « Linux CPU », « Linux Memory », etc. ?

➡️ **Normal sur Zabbix 7.4.**
Tous les templates sont intégrés dans **un seul template moderne**.

Pour l’ajouter :

```
Linked templates → Select → "Linux by Zabbix agent"
```

Valider → **Update**

---

# 4. **Création d’un Dashboard**

Menu :

```
Monitoring → Dashboards → Create dashboard
```

Nom :
`Supervision Linux - Atelier`

---

## 📊 Ajouter des widgets utiles

### CPU

```
Graph → system.cpu.util[]
```

### Mémoire

```
vm.memory.size[available]
vm.memory.size[total]
```

### Disques

```
vfs.fs.size[/,pused]
```

### Réseau

```
net.if.in[eth0]
net.if.out[eth0]
```

### État agent

```
agent.ping
```

Résultat :
Un dashboard complet permettant de suivre l’état global d’un serveur Linux.

---

# 5. **Dépannage**

### ❌ Agent unreachable (3m)

* Hostname erroné
* Port 10050 bloqué
* Agent non démarré
* Mauvais Server / ServerActive

### ❌ No data

Ouvrir le port :

```bash
ufw allow 10050/tcp
```

### ❌ ICMP ne fonctionne pas

* fping absent
* mauvaise permission (`u+s`)

### ❌ Cannot connect to [[127.0.0.1]:10051]

* Mauvais `ServerActive`
* Firewall bloquant

---


