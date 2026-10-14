# 🛡️ Étude et mise en place d'un système de prévention d'intrusion de nouvelle génération (NGIPS) - pfSense, Snort, ELK
---
## 📑 Table des matières



4. [Architecture de la solution](#4-architecture-de-la-solution)
5. [Environnement de test](#5-environnement-de-test)
6. [Mise en œuvre](#6-mise-en-œuvre)
7. [Tests d'intrusion et résultats](#7-tests-dintrusion-et-résultats)
8. [Supervision avec la suite ELK](#8-supervision-avec-la-suite-elk)
10. [Conclusion et perspectives](#10-conclusion-et-perspectives)
11. [Références](#11-références)
12. [Liste des figures](#12-liste-des-figures)


![Positionnement d'un pare-feu dans un réseau](images/figure-3-1.png)
*Positionnement standard d'un pare-feu dans une architecture réseau*

![Système de détection d'intrusion (IDS)](images/figure-3-6.png)
*Position d'un IDS dans un réseau*

![Système de prévention d'intrusion (IPS)](images/figure-3-7.png)
*Fonctionnement d'un IPS*

Types d'IPS selon la position : **NIPS** (Network IPS, protège l'ensemble du réseau) et **HIPS** (Host IPS, protège un hôte spécifique).

> Architecture classique : pare-feu puis IPS en équipements distincts. Architecture retenue ici : **IDS/IPS intégré nativement au pare-feu** (Snort au sein de pfSense), donnant un NGFW/NGIPS en un seul équipement — réduction du coût et de la complexité de gestion.

---

## 4. Architecture de la solution

 Une **architecture réseau de test** reproduisant l'essentiel du réseau réel a été conçue :

![Architecture réseau intégrant le NGIPS et ELK](images/figure-4-1.jpeg)
*Architecture réseau de test — pfSense + Snort (NGIPS) et supervision ELK*

| Élément | Rôle | Adressage |
|---|---|---|
| Pare-feu pfSense (NGIPS) | Filtrage + IDS/IPS (Snort intégré) | WAN : interface vers Internet · LAN : `192.168.0.1/16` |
| Machine attaquante (Kali) | Simulation d'attaques, sur le WAN | `192.168.222.131/24` |
| Switch | Interconnexion LAN | — |
| Machines clientes (Windows) | Postes supervisés | `192.168.1.105/24`, `192.168.1.106/24` |
| Serveur NGIPS / ELK | Centralisation et visualisation des logs | `192.168.222.138/24` |

La machine attaquante est positionnée sur le réseau WAN (Internet), simulant un intrus externe. Les logs des postes Windows sont envoyés vers la suite ELK pour analyse.

---

## 5. Environnement de test

Travaux réalisés en environnement **entièrement virtualisé** (GNS3 + VMware Workstation 17), sur un poste HP ProBook 435 G8 (AMD Ryzen 7 PRO 5850U, 32 Go RAM, 512 Go SSD, Windows 11 Pro, virtualisation activée).

| VM | Rôle | Mémoire | vCPU | Stockage | Système |
|---|---|---|---|---|---|
| **VM1** | Suite ELK (visualisation logs Windows + pfSense) | 4 Go | 2 | 50 Go | Ubuntu Server 64 bits |
| **VM2** | Machine cliente supervisée | 4 Go | 4 | 30 Go | Windows 10 64 bits |
| **VM3** | Machine attaquante (tests) | 2 Go | 2 | 40 Go | Kali Linux 64 bits |

---

## 6. Mise en œuvre

### 6.1 Configuration réseau et installation de pfSense

![Création des cartes réseaux](images/figure-4-6.png)
*Création du réseau virtuel VMNet3 et de l'interface NAT*

![Interfaces WAN et LAN](images/figure-4-7.png)
*Attribution des réseaux NAT (WAN) et VMNet3 (LAN)*

![Test de connectivité](images/figure-4-8.png)
*Test d'accès Internet depuis pfSense (ping vers 8.8.8.8, DNS Google)*

![Vérification de l'adressage du serveur](images/figure-4-9.png)
*Vérification de l'adressage de la machine cliente Ubuntu Desktop*

![Création de l'interface DMZ](images/figure-4-10.png)
*Ajout de la troisième interface réseau (DMZ) sur pfSense*

![Connexion à PfSense](images/figure-4-11.png)
*Connexion à l'interface web de pfSense (identifiants par défaut modifiés : admin / pfsense d'origine)*

![Tableau de bord PfSense](images/figure-4-12.png)
*Tableau de bord pfSense avec les 3 interfaces (WAN, LAN, DMZ) actives*

### 6.2 Installation et configuration de Snort

Installation via **Système → Gestionnaire de paquets → Paquets disponibles → recherche « snort » → Install**.

![Installation de Snort](images/figure-4-13.png)
*Installation du package Snort sur pfSense*

![Accès à l'interface de Snort](images/figure-4-14.png)
*Accès au menu Services → Snort*

![Paramètres généraux de Snort](images/figure-4-15.png)
*Activation des règles Snort GPLv2 Community et ET Open (Emerging Threats)*

![Paramètres de mises à jour](images/figure-4-16.png)
*Paramètres de mise à jour des règles*

#### Activation sur les interfaces

L'option **« Block Offenders »** a été activée sur chaque interface afin que Snort fonctionne réellement comme un **IPS** (blocage actif) et non comme un simple IDS passif.

![Paramètres de l'interface WAN](images/figure-4-17.png)
*Ajout de l'interface WAN dans Snort Interfaces*

![Règles de l'interface WAN](images/figure-4-18.png)
*Activation des règles sur l'interface WAN*

![Règles WAN Catégories](images/figure-4-19.png)
*Ajustement des catégories de règles WAN*

![Activation des règles de Snort](images/figure-4-20.png)
*Activation de l'ensemble des règles Snort*

![Activation des interfaces sur Snort](images/figure-4-21.png)
*Snort actif sur les 3 interfaces : WAN (intrusions externes), LAN (activités internes), DMZ (trafic DMZ)*

---

## 7. Tests d'intrusion et résultats

Trois attaques ont été lancées depuis la machine Kali Linux (VM3, positionnée sur le WAN) vers la machine cible Ubuntu (`192.168.1.104`), afin de valider le comportement **détection + blocage** de Snort.

### 7.1 Scan de ports (Nmap)

```bash
nmap 192.168.1.104
```

![Scan Nmap](images/figure-4-22.png)
*Lancement du scan de ports avec Nmap*

![Détection des alertes Nmap](images/figure-4-23.png)
*Alerte Snort (interface WAN) — IP source détectée : 192.168.222.131, type « SCAN Suspicious… »*

![Tentatives Nmap bloqués](images/figure-4-24.png)
*Confirmation du blocage de l'adresse 192.168.222.131 (onglet Blocked)*

**Résultat :** détection ✅ — blocage ✅

### 7.2 Attaque par déni de service (DoS — inondation SYN)

```bash
hping3 --flood --rand-source -S -p 22 192.168.1.104
```

Envoi massif de paquets SYN vers le port 22 de la cible, avec adresses IP source aléatoires, visant à épuiser les ressources réseau.

![Attaque DoS](images/figure-4-25.png)
*Lancement de l'attaque DoS avec hping3*

![Détection de l'attaque DoS](images/figure-4-26.png)
*Détection de l'attaque par Snort*

![Blocage de l'attaque DoS](images/figure-4-27.png)
*Blocage de l'attaque par Snort*

**Résultat :** détection ✅ — blocage ✅

### 7.3 Attaque par force brute (SSH)

```bash
hydra -l NGiPS -P /usr/share/wordlists/rockyou.txt.gz ssh://192.168.1.104
```

Tentative de connexion SSH par force brute avec l'outil **Hydra**, utilisateur `NGIPS`, dictionnaire `rockyou.txt.gz`.

![Attaque par brute force](images/figure-4-28.png)
*Lancement de l'attaque par force brute avec Hydra — réponse « No route to host » (connexion bloquée)*

![Détection de la tentative de connexion par brute force](images/figure-4-29.png)
*Détection de la tentative par Snort*

![Blocage de la tentative de connexion par brute force](images/figure-4-30.png)
*Blocage de la tentative par Snort*

**Résultat :** détection ✅ — blocage ✅ (l'erreur « No route to host » confirme le blocage réseau)

### 7.4 Synthèse des tests

| Attaque | Outil | Cible | Détection Snort | Blocage Snort |
|---|---|---|---|---|
| Scan de ports | Nmap | 192.168.1.104 | ✅ | ✅ |
| DoS (flood SYN) | hping3 | 192.168.1.104:22 | ✅ | ✅ |
| Force brute SSH | Hydra | 192.168.1.104:22 | ✅ | ✅ |

---

## 8. Supervision avec la suite ELK

La suite **ELK (Elasticsearch, Logstash, Kibana)**, développée par Elastic, assure la collecte, le stockage, l'analyse et la visualisation des logs ; combinée à des règles de détection/corrélation, elle peut fonctionner comme un **SIEM**.

### 8.1 Installation d'Elasticsearch et Kibana (sur VM1 — Ubuntu Server)

```bash
# Prérequis
apt update && apt-get install apt-transport-https gnupg curl wget

# Clé PGP Elasticsearch
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

# Dépôt Elastic
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | tee /etc/apt/sources.list.d/elastic-8.x.list

# Installation Elasticsearch
apt update && apt-get install elasticsearch
```

![Installation d'Elasticsearch](images/figure-4-31.png)
*Confirmation du succès de l'installation d'Elasticsearch*

```bash
# Installation Kibana
apt update && apt-get install kibana
```

Configuration :

```yaml
# /etc/elasticsearch/elasticsearch.yml
network.host: 192.168.222.138
```

```bash
systemctl start elasticsearch.service
```

```yaml
# /etc/kibana/kibana.yml
server.host: "0.0.0.0"
server.publicBaseUrl: "http://192.168.222.138:5601"
```

```bash
systemctl start kibana.service
```

Connexion sécurisée obtenue à l'adresse `https://192.168.222.138:5601` :

![Page de connexion Kibana](images/figure-4-32.png)
*Interface de connexion à Kibana*

### 8.2 Installation et configuration de l'agent Winlogbeat (sur VM2 — Windows)

**Winlogbeat** collecte les événements du journal Windows (connexions, tentatives échouées, comptes bloqués…) et les envoie vers Elasticsearch. Téléchargement : <https://www.elastic.co/downloads/beats/winlogbeat>

Configuration du fichier `winlogbeat.yml` avec l'adresse du serveur :

```yaml
setup.kibana:
  host: "https://192.168.222.138:5601"
  ssl.verification_mode: none
  username: "elastic"
  password: "<mot_de_passe_elastic>"

output.elasticsearch:
  hosts: ["192.168.222.138:9200"]
  protocol: "https"
  ssl.verification_mode: none
  username: "elastic"
  password: "<mot_de_passe_elastic>"
```

![Configuration de Winlogbeat](images/figure-4-33.png)
*Extrait du fichier winlogbeat.yml — sections Kibana et Elasticsearch Output*

Démarrage du service :

```powershell
PS C:\> Start-Service winlogbeat
```

![Vérification de Winlogbeat](images/figure-4-34.png)
*Vérification de la communication Winlogbeat → ELK (« Data successfully received »)*

### 8.3 Test : détection d'une authentification non légitime

Scénario : tentative de connexion à la session utilisateur **« User »** avec un mot de passe erroné, pour observer la réaction de la supervision.

![Tentative de connexion erronée](images/figure-4-35.png)
*Tentative de connexion avec mot de passe erroné sur la machine Windows*

![Connexion rejetée](images/figure-4-36.png)
*Connexion rejetée côté Windows*

![Détection des activités sur Windows](images/figure-4-37.png)
*Dashboard Winlogbeat dans ELK — mise à jour automatique des « Failed Logons », colonne `event.action` avec l'heure et l'utilisateur concerné*

**Résultat attendu et observé :**
- Alerte affichée sur le Dashboard Winlogbeat de la suite ELK ✅
- Journalisation de l'événement (utilisateur, heure, type d'échec) ✅

---

## 10. Conclusion et perspectives

La solution implémentée — **pfSense** (filtrage réseau) + **Snort IDS/IPS** (détection et prévention, intégré nativement au pare-feu) + **Winlogbeat** (collecte des journaux Windows) + **suite ELK** (analyse et visualisation en temps réel) — s'est montrée capable de :

- détecter et bloquer automatiquement plusieurs types d'attaques (scan de ports, DoS, force brute SSH) ;
- superviser en continu les activités réseau et les postes clients Windows ;
- centraliser les journaux et alertes sur un tableau de bord unique.

**Perspective d'amélioration identifiée :** intégration d'un **VPN site à site** pour renforcer la sécurisation des communications distantes.

---

## 11. Références

Sélection des références techniques citées dans le mémoire (liste complète dans le document original) :

| Sujet | Source |
|---|---|
| Positionnement et principe des pare-feux | Cisco — <https://www.cisco.com/site/fr/fr/learn/topics/security/what-is-a-firewall.html> |
| Types de pare-feux | Geekflare — <https://geekflare.com/fr/types-of-firewalls/> |
| pfSense | Netgate — <https://www.netgate.com/> · Documentation : <https://docs.netgate.com/> |
| FortiGate NGFW | Fortinet — <https://www.fortinet.com/products/next-generation-firewall> |
| OPNsense | <https://opnsense.org/> |
| Snort | Snort Project — <https://snort.org/> |
| Suricata | OISF — <https://suricata.io/> |
| NGIPS | Cisco — <https://www.cisco.com/c/en_ca/products/security/ngips/index.html> |
| GNS3 | <https://www.gns3.com/> |
| VMware Workstation/Fusion | <https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion> |
| Suite Elastic (ELK) | <https://www.elastic.co> · Documentation : <https://www.elastic.co/guide> |
| Analyse Snort 3 (Cisco) | <https://www.cisco.com/c/en/us/td/docs/security/secure-firewall/management-center/snort3/...> |
| HOMINTEC | <https://homintec.com/> |

---

## 12. Liste des figures retenues

| Figure | Titre | Fichier |
|---|---|---|
| 3.1 | Positionnement d'un pare-feu dans un réseau informatique | `images/figure-3-1.png` |
| 3.6 | Système de détection d'intrusion (IDS) | `images/figure-3-6.png` |
| 3.7 | Système de prévention d'intrusion (IPS) | `images/figure-3-7.png` |
| 4.1 | Architecture réseau intégrant le NGIPS et ELK | `images/figure-4-1.jpeg` |
| 4.5 | Interaction Winlogbeat et ELK | `images/figure-4-5.png` |
| 4.6 | Création des cartes réseaux | `images/figure-4-6.png` |
| 4.7 | Interfaces WAN et LAN | `images/figure-4-7.png` |
| 4.8 | Test de connectivité | `images/figure-4-8.png` |
| 4.9 | Vérification de l'adressage du serveur | `images/figure-4-9.png` |
| 4.10 | Création de l'interface DMZ | `images/figure-4-10.png` |
| 4.11 | Connexion à PfSense | `images/figure-4-11.png` |
| 4.12 | Tableau de bord PfSense | `images/figure-4-12.png` |
| 4.13 | Installation de Snort | `images/figure-4-13.png` |
| 4.14 | Accès à l'interface de Snort | `images/figure-4-14.png` |
| 4.15 | Paramètres généraux de Snort | `images/figure-4-15.png` |
| 4.16 | Paramètres de mises à jour | `images/figure-4-16.png` |
| 4.17 | Paramètres de l'interface WAN | `images/figure-4-17.png` |
| 4.18 | Règles de l'interface WAN | `images/figure-4-18.png` |
| 4.19 | Règles WAN Catégories | `images/figure-4-19.png` |
| 4.20 | Activation des règles de Snort | `images/figure-4-20.png` |
| 4.21 | Activation des interfaces sur Snort | `images/figure-4-21.png` |
| 4.22 | Scan Nmap | `images/figure-4-22.png` |
| 4.23 | Détection des alertes Nmap | `images/figure-4-23.png` |
| 4.24 | Tentatives Nmap bloqués | `images/figure-4-24.png` |
| 4.25 | Attaque DoS | `images/figure-4-25.png` |
| 4.26 | Détection de l'attaque DoS | `images/figure-4-26.png` |
| 4.27 | Blocage de l'attaque DoS | `images/figure-4-27.png` |
| 4.28 | Attaque par brute force | `images/figure-4-28.png` |
| 4.29 | Détection de la tentative de connexion par brute force | `images/figure-4-29.png` |
| 4.30 | Blocage de la tentative de connexion par brute force | `images/figure-4-30.png` |
| 4.31 | Installation d'Elasticsearch | `images/figure-4-31.png` |
| 4.32 | Page de connexion Kibana | `images/figure-4-32.png` |
| 4.33 | Configuration de Winlogbeat | `images/figure-4-33.png` |
| 4.33b | Démarrage du service Winlogbeat (PowerShell) | `images/figure-4-33b-start-service.jpg` |
| 4.34 | Vérification de Winlogbeat | `images/figure-4-34.png` |
| 4.35 | Tentative de connexion erronée | `images/figure-4-35.png` |
| 4.36 | Connexion rejetée | `images/figure-4-36.png` |
| 4.37 | Détection des activités sur Windows | `images/figure-4-37.png` |

> Les logos des produits présentés (Cisco ASA, PfSense, Fortinet, OPNsense, Snort, Suricata, GNS3, VMware, ELK) ainsi que les figures institutionnelles (localisation et organigrammes de l'EPL et de HOMINTEC) n'ont pas été repris ici, conformément à la demande de ne garder que les éléments techniques pertinents.
