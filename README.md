# Projet Fil Rouge NetDevOps — Infrastructure Automatisée, Sécurisée et Supervisée

Ce dépôt contient l'ensemble du code, des configurations d'infrastructure et des pipelines CI/CD du projet **NetDevOps**. L'objectif est de concevoir, déployer, sécuriser et superviser une infrastructure réseau segmentée en trois zones à l'aide des pratiques **Infrastructure as Code (IaC)** et **DevSecOps**.

---

## 1. Architecture Réseau et Segmentation

L'infrastructure repose sur un hôte Linux hébergeant Docker et trois réseaux virtuels isolés, interconnectés et filtrés par un pare-feu central (`router-fw`).

```text
                      +-----------------------------------+

                      |         Usine Logicielle          |
                      |  GitHub -> Jenkins -> Trivy Scan  |
                      +-----------------+-----------------+
                                        |
                                        v
+-------------------------------------------------------------------------------+

|                           Hôte Linux (Docker Engine)                          |
|                                                                               |
|  +-----------------------+     +-------------------+     +-----------------+  |
|  |   Zone DMZ            |     |   Routeur / FW    |     |   Zone LAN      |  |
|  |   172.20.1.0/24       |<--->|   router-fw       |<--->|   172.20.2.0/24 |  |
|  |                       |     |   (nftables)      |     |                 |  |
|  | - web-dmz (Nginx)     |     +---------+---------+     | - dns-lan       |  |
|  |   172.20.1.10:80      |               |               |   (dnsmasq)     |  |
|  +-----------------------+               |               | - client1 / 2   |  |
|                                          |               +-----------------+  |
|                                          v                                    |
|                            +---------------------------+                      |
|                            |   Zone Management         |                      |
|                            |   172.20.3.0/24           |                      |
|                            |                           |                      |
|                            | - Prometheus (:9090)      |                      |
|                            | - Grafana (:3000)         |                      |
|                            | - Node Exporter (:9100)   |                      |
|                            +---------------------------+                      |
+-------------------------------------------------------------------------------+
```

### Inventaire des services et adressage IP

| Zone Réseau | Sous-réseau | Conteneur | Rôle technique | Ports exposés |
| :--- | :--- | :--- | :--- | :--- |
| **DMZ** | `172.20.1.0/24` | `web-dmz` | Serveur Web et Reverse Proxy Nginx durci | `8080:80` |
| **Routage / Sécurité** | Multi-zones | `router-fw` | Routeur inter-VLAN et pare-feu `nftables` | — |
| **LAN** | `172.20.2.0/24` | `dns-lan` | Résolution DNS interne (`*.pme.local`) via `dnsmasq` | `53/tcp`, `53/udp` |
| **LAN** | `172.20.2.0/24` | `client1` / `client2` | Postes clients Alpine Linux pour validation des flux | — |
| **Management** | `172.20.3.0/24` | `monitoring` | Collecte de métriques système et réseau (Prometheus) | `9090:9090` |
| **Management** | `172.20.3.0/24` | `grafana` | Tableaux de bord avec provisioning JSON automatique | `3000:3000` |
| **Management** | `172.20.3.0/24` | `node-exporter` | Exportation des métriques matérielles de l'hôte | `9100/tcp` |
| **CI/CD** | Hôte / Bridge | `jenkins` | Orchestrateur d'intégration et de sécurité continue | `8081:8080` |

---

## 2. Arborescence du Dépôt

```text
netdevops/
├── ansible/                  # Playbooks, inventaires et rôles IaC (nftables, dnsmasq, nginx_proxy)
├── docker/
│   ├── dns-lan/              # Configuration et Dockerfile du serveur DNS interne
│   ├── grafana/              # Provisioning automatique et dashboard JSON (NetDevOps - Supervision)
│   ├── jenkins/              # Image personnalisée Jenkins avec Docker CLI et Python
│   ├── monitoring/           # Configuration des cibles de collecte Prometheus
│   ├── router-fw/            # Règles de filtrage nftables et routage IP
│   └── web-dmz/              # Configuration Nginx durcie (en-têtes de sécurité, reverse proxy)
├── reports/                  # Rapports générés (tests de connectivité, audits)
├── tests/
│   └── connectivity.sh       # Suite de tests automatisés de connectivité, DNS et filtrage
├── docker-compose.yml        # Déploiement de l'infrastructure réseau et supervision
├── docker-compose.tools.yml  # Déploiement de l'usine logicielle (Jenkins)
├── Jenkinsfile               # Définition du pipeline CI/CD et DevSecOps
├── .yamllint                 # Règles de validation syntaxique YAML
└── README.md                 # Documentation technique du projet
```

---

## 3. Sécurité et Durcissement (Hardening)

* **Cloisonnement réseau strict (`nftables`)** :
  * Autorisation des flux légitimes depuis le **LAN** vers le serveur web en **DMZ** (`TCP/80`).
  * Blocage explicite des connexions initiées depuis la **DMZ** vers le **LAN** (prévention des mouvements latéraux).
  * Filtrage de l'administration **SSH** (`TCP/22`) sur le routeur.
* **Durcissement Applicatif (Nginx)** :
  * Masquage de la version du serveur (`server_tokens off`).
  * Ajout des en-têtes de sécurité HTTP (`X-Frame-Options`, `X-Content-Type-Options`).
* **Gestion des secrets** :
  * Exclusion stricte des fichiers sensibles (`.env`, `.vault_pass`) via `.gitignore`.
  * Chiffrement des variables sensibles via **Ansible Vault**.

---

## 4. Pipeline CI/CD & DevSecOps (Jenkins)

Le fichier `Jenkinsfile` automatise la validation continue à chaque `push` sur la branche `develop` :

1. **Vérification du dépôt** : Contrôle de l'intégrité de l'arborescence et de la présence des manifests.
2. **Qualité du code (Lint)** : Exécution de `yamllint` et `ansible-lint` via un conteneur éphémère Python.
3. **Build Docker** : Construction automatisée des images personnalisées (`netdevops/*:1.0`).
4. **Analyse de vulnérabilités (Trivy)** : Scan de sécurité des images Docker (`web-dmz`, `dns-lan`, `router-fw`) avec blocage en cas de faille critique ou élevée non corrigée (`--severity HIGH,CRITICAL`).

---

## 5. Guide de Déploiement Rapide

### Prérequis
* Serveur Linux (Ubuntu 22.04/24.04) avec **Docker Engine** et **Docker Compose v2**.
* Ports disponibles sur l'hôte : `3000` (Grafana), `8080` (Web DMZ), `8081` (Jenkins), `9090` (Prometheus).

### Lancement de l'infrastructure et des outils
```bash
# 1. Cloner le dépôt
git clone https://github.com/Stvrk906/netdevops.git
cd netdevops

# 2. Démarrer l'infrastructure réseau et la supervision
docker compose up -d --build

# 3. Démarrer l'usine logicielle Jenkins
docker compose -f docker-compose.tools.yml up -d --build
```

### Exécution des tests de validation
```bash
bash tests/connectivity.sh
```

---

## 6. Accès aux Interfaces Web

* **Application Web (DMZ)** : `http://<IP_SERVEUR>:8080`
* **Tableau de bord Grafana** : `http://<IP_SERVEUR>:3000` *(Dossier : NetDevOps > NetDevOps - Supervision)*
* **Prometheus** : `http://<IP_SERVEUR>:9090`
* **Pipeline Jenkins** : `http://<IP_SERVEUR>:8081`
