# 📊 Monitoring Stack -- Zabbix + Grafana + NGINX

Infrastructure de supervision basée sur :

-   🐳 Docker\
-   🗄 PostgreSQL\
-   📡 Zabbix Server\
-   📈 Grafana\
-   🔐 NGINX Reverse Proxy (HTTPS)

------------------------------------------------------------------------

## 🎯 Objectif

Déployer une stack de supervision :

-   Portable\
-   Versionnable\
-   Paramétrable via `.env`\
-   Adaptée à un environnement entreprise\
-   Sans valeur sensible hardcodée

------------------------------------------------------------------------

## 🏗 Architecture

``` mermaid
flowchart LR

    subgraph CLIENTS
        AGENT[Zabbix Agents]
    end

    subgraph DMZ
        NGINX[NGINX Reverse Proxy]
    end

    subgraph SERVERS
        ZWEB[Zabbix Web]
        ZSRV[Zabbix Server]
        DB[(PostgreSQL)]
        GRAFANA[Grafana]
    end

    AGENT -->|10050| ZSRV
    ZSRV --> DB
    ZWEB --> ZSRV
    NGINX -->|HTTPS| ZWEB
    NGINX -->|HTTPS| GRAFANA
```

------------------------------------------------------------------------

## 📂 Structure du projet

.
├── docker-compose.yml
├── README.md
└── reverse-proxy
    ├── certs
    ├── conf.d
    └── templates
        └── default.conf

------------------------------------------------------------------------

## 🐳 Installation de Docker (Debian)

``` bash
set -e

sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release

sudo mkdir -p /usr/share/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | \
sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
"deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
https://download.docker.com/linux/debian $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

sudo systemctl enable docker
sudo systemctl start docker

sudo docker run hello-world
```

------------------------------------------------------------------------

## 📥 Cloner le projet

Cloner le dépôt :

```bash
git clone https://github.com/remiwbk/zabbix-grafana-nginx.git
```

------------------------------------------------------------------------

## ⚙️ Configuration

### 1️⃣ Copier le fichier d'exemple

``` bash
cp .env.example .env
```

Modifier les variables :

``` env
ZABBIX_DOMAIN=zabbix.red.home
GRAFANA_DOMAIN=grafana.red.home
CERT_NAME=zabbix
HTTP_PORT=80
HTTPS_PORT=443
```
------------------------------------------------------------------------

## 🧩 NGINX – Variables dynamiques avec envsubst

NGINX ne lit pas directement les variables du `.env`.  
Nous utilisons `envsubst` au démarrage du conteneur pour générer la configuration finale.

### 📁 Structure

```
reverse-proxy/
├── certs
├── conf.d          # vide côté hôte
└── templates
    └── default.conf   # contient ${VARIABLE}
```

### 🔄 Fonctionnement

- `templates/default.conf` → contient les variables `${...}`
- `envsubst` remplace les variables au démarrage
- Le fichier final est généré dans `/etc/nginx/conf.d/default.conf`

⚠️ Ne pas placer de `default.conf` dans `conf.d/` côté hôte.

------------------------------------------------------------------------

### 2️⃣ Génération des certificats TLS (auto-signé)

``` bash
mkdir -p reverse-proxy/certs

openssl req -x509 -nodes -days 365 \
-newkey rsa:2048 \
-keyout reverse-proxy/certs/zabbix.key \
-out reverse-proxy/certs/zabbix.crt \
-subj "/CN=zabbix.red.home"
```

Les fichiers doivent être :

    reverse-proxy/certs/${CERT_NAME}.crt
    reverse-proxy/certs/${CERT_NAME}.key

------------------------------------------------------------------------

### 3️⃣ Lancer la stack

``` bash
docker compose up -d
```

Vérifier :

``` bash
docker ps
```

------------------------------------------------------------------------

## 🐳 Particularité Docker – IP du Zabbix Server

Lorsque Zabbix Server est exécuté dans Docker, l’interface du host **Zabbix server** dans l’UI peut nécessiter l’IP interne du conteneur (réseau bridge Docker), et non `127.0.0.1` ni l’IP de la machine hôte.

Vérifier l’IP interne :

```bash
docker inspect zabbix-server | grep IPAddress
```
⚠️ Cette IP peut changer si le conteneur est recréé.

------------------------------------------------------------------------

## 🔐 Sécurité

-   Reverse proxy HTTPS\
-   Redirection HTTP → HTTPS\
-   Paramétrage dynamique via `.env`\
-   Aucune donnée sensible dans le dépôt

------------------------------------------------------------------------

## 📡 Ports utilisés

  Service         Port
  --------------- -------
  Zabbix Server   10051
  Zabbix Agent    10050
  HTTP            80
  HTTPS           443

------------------------------------------------------------------------

## 🚀 Améliorations possibles

-   Provisioning automatique Grafana\
-   Déploiement via Ansible\
-   Infrastructure as Code (Terraform)\
-   Let's Encrypt\
-   Healthchecks Docker\
-   Haute disponibilité

------------------------------------------------------------------------

## 👨‍💻 Auteur

Projet réalisé dans le cadre d'une montée en compétence en
administration système, supervision et industrialisation
d'infrastructure.
