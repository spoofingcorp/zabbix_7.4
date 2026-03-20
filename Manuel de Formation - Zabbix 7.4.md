# **Manuel de Formation Complet \- Zabbix 7.4**

## **1\. Introduction à la Formation**

### **1.1. Objectifs Pédagogiques**

À la fin de cette formation, les participants seront capables de :

* Comprendre l'architecture complexe et les flux de données de Zabbix 7.0.  
* Installer et configurer un serveur Zabbix 7.0 sur Ubuntu 24.04 (PostgreSQL/Nginx).  
* Déployer et configurer Zabbix Agent et Agent 2 (passif/actif) sur Ubuntu 24.04 et Windows Server 2025\.  
* Superviser un équipement réseau (Switch Cisco) via SNMP, y compris la découverte LLD.  
* Maîtriser la création d'Items, de Triggers complexes et de dépendances.  
* Mettre en place un système d'alertes (Actions) efficace et éviter les "flapping".  
* Créer des visualisations métiers (Dashboards, Maps).  
* Comprendre (HA, SLA, Corrélation).

### **1.2. Public Cible**

Administrateurs systèmes, ingénieurs réseau, et toute personne responsable de la supervision d'une infrastructure informatique.

### **1.3. Prérequis**

* Bonnes connaissances de l'administration système Linux (ligne de commande, services).  
* Connaissances de base de Windows Server (installation de services).  
* Compréhension des concepts réseau (TCP/IP, adressage IP, ports).  
* Notions de base sur le protocole SNMP (un rappel sera fait).

### **1.4. Environnement Technique du Laboratoire**

* **Serveur Zabbix:** Zabbix 7.0 LTS sur Ubuntu Server 24.04 (Noble Numbat).  
* **Base de données:** PostgreSQL.  
* **Serveur Web:** Nginx.  
* **Cibles de supervision:**  
  * 1x Windows Server 2025 (Zabbix Agent 2).  
  * 1x Ubuntu Server 24.04 (Zabbix Agent 2).  
  * 1x Switch Cisco (SNMPv2c ou v3).

## **Jour 1 : Fondamentaux et Installation**

### **Matin : Théorie \- Les Piliers de Zabbix**

#### **Module 1 : Introduction à la Supervision**

**1.1. Les Objectifs de la Supervision** Pourquoi superviser ? La supervision n'est pas seulement "vérifier si ça marche". Elle répond à trois grands besoins métier :

1. **Garantir la Disponibilité (Proactivité) :**  
   * **Avant la panne :** Détecter les signes avant-coureurs (disque plein à 80%, CPU élevé pendant 10 min) pour agir *avant* que le service ne tombe.  
   * **Pendant la panne :** Alerter la bonne équipe, au bon moment, avec la bonne information (diagnostiquer la cause racine).  
   * **Après la panne :** Fournir les données pour l'analyse post-mortem (comprendre ce qui s'est passé).

   

2. **Mesurer la Performance :**  
   * Suivre les temps de réponse d'un site web, le débit d'une interface réseau, le nombre de transactions par seconde d'une base de données.  
   * Permet de valider que les services délivrés respectent les attentes (performances applicatives).

   

3. **Gérer la Capacité (Capacity Planning) :**  
   * En observant les tendances (tendances), on peut anticiper les besoins futurs.  
   * *Exemple :* "Au rythme actuel, le stockage de ce serveur sera plein dans 3 mois."  
   * Permet de justifier et planifier les investissements matériels ou les migrations cloud.

   

**1.2. Présentation et Historique de Zabbix**

* **Créateur :** Alexei Vladishev (Letton).  
* **Historique :**  
  * **1998 :** Début du projet comme un logiciel interne.  
  * **2001 :** Publication en Open Source.  
  * **2004 :** Sortie de Zabbix 1.0 stable.  
  * **2005 :** Création de la société Zabbix SIA.


* **Modèle Économique :** 100% Open Source. Le logiciel (serveur, agent) est entièrement gratuit, sans version "Entreprise" bridée. L'entreprise Zabbix se finance par le support professionnel, la formation, et le développement.

* **Versions :**  
  * **LTS (Long Term Support) :** Supportées 5 ans (ex: 5.0, 6.0, 7.0). À privilégier en production.  
  * **Standard :** Supportées 6 mois (ex: 6.2, 6.4). Introduisent les nouveautés.


* **Zabbix 7.0 LTS (Avril 2024\) :** Apporte des améliorations majeures sur la performance (backend Go), la haute disponibilité, les dashboards (widgets), et l'intégration (webhooks).

* **Ressources :**   
    [Documentation officielle](https://www.zabbix.com/documentation/current/en/manual/quickstart/host)  
    [Zabbix Share (partage de templates)](https://www.zabbix.com/fr/integrations)  
    [Zabbix Community (forum)](https://www.zabbix.com/forum)

  [Download Zabbix / Agent](https://www.zabbix.com/download?zabbix=7.2&os_distribution=ubuntu&os_version=24.04&components=agent&db=&ws=) 

#### **Module 2 : Architecture Zabbix 7.0**

![pix/z1.png](pix/z1.png)


* **Zabbix Server (zabbix\_server) :**

  * Le cerveau. Écrit en Go (depuis la 7.0, anciennement en C).


  * **Rôles :**  
    * **Pollers :** Processus qui collectent les données (agents passifs, SNMP, JMX, IPMI).  
    * **Trappers :** Processus qui écoutent (sur le port 10051\) les données envoyées par les agents actifs ou les proxies.  
    * **History Syncer :** Écrit les données collectées dans la base de données.  
    * **Timer :** Gère le temps, les maintenances, les actions.  
    * **Alerters :** Gèrent l'envoi des notifications (email, scripts, webhooks).

![pix/z2.png](pix/z2.png)


**Bonne Pratique :** Le serveur Zabbix ne doit faire *que* Zabbix. Ne pas installer d'autres services (serveur web, mail) dessus pour des raisons de performance.

* **Base de Données (PostgreSQL / TimescaleDB) :**  
  * Le cœur du stockage.  
  * **Stocke quoi ?**  
    * **Configuration :** Hôtes, Items, Triggers, Maps... (tout sauf l'historique).  
    * **Historique :** Les données brutes (ex: 15.2% CPU à 10:01).  
    * **Tendances :** Les données agrégées (moyenne par heure) pour garder une vision long terme sans saturer la base.

    

  **Bonne Pratique :** Utiliser **PostgreSQL** (recommandé) avec **TimescaleDB**. TimescaleDB est une extension qui optimise PostgreSQL pour les séries temporelles, résultant en des performances 10x à 100x supérieures pour les graphiques et le "housekeeping".


* **Frontend Web (Nginx \+ PHP) :**  
  * L'interface de configuration et de visualisation.  
  * Communique avec le Zabbix Server via l'API Zabbix et lit/écrit dans la base de données (pour la configuration).


* **Zabbix Agent (Agent) :**  
  * Le collecteur local sur les hôtes (Windows, Linux). Détaillé au Module 3\.


* **Zabbix Proxy (zabbix\_proxy) :**  
  * Un "mini" serveur Zabbix déporté. Il collecte les données de ses agents, les stocke localement (dans une base SQLite ou PostgreSQL), puis les envoie en *batch* au Zabbix Server principal.


  * **Cas d'usage (REX) :**


    * **Supervision distribuée :** Indispensable pour superviser des sites distants (agences bancaires, magasins) connectés par un WAN. Un proxy par site limite le trafic réseau à une seule connexion vers le central.

    

    * **Performance :** Si vous avez plus de 5 000 hôtes, le serveur Zabbix principal devient un goulot d'étranglement. On utilise des proxies pour répartir la charge de collecte, même s'ils sont dans le même datacenter.

    

    * **Réseaux isolés (DMZ) :** On place un proxy en DMZ pour collecter les données des serveurs publics. Le proxy est le seul à initier la connexion vers le Zabbix Server (en mode actif), gardant le flux DMZ \-\> LAN sécurisé et maîtrisé.

#### 

#### **Module 3 : Concepts Clés (Le Vocabulaire Zabbix)**

![pix/z2.png](pix/z2.png)

* **Host (Hôte) :** L'équipement à superviser (un serveur, un switch, une VM).

* **Host Group (Groupe d'hôtes) :** Un conteneur logique (ex: "Serveurs Linux", "Switchs Cisco", "Serveurs de Production").

* **Item (Métrique) :** L'élément de donnée collecté (ex: system.cpu.load). C'est la *question*.  
  * Il définit : Quoi collecter ? (la clé), À quelle fréquence ? (Update interval), Combien de temps garder l'historique ?


* **Trigger (Déclencheur) :** L'expression logique qui définit un problème. C'est le *seuil d'alerte*.  
  * *Exemple :* last(/MonServeur/system.cpu.load) \> 5


* **Problem (Problème) & Event (Événement) :**  
  * Un **Événement** est un changement d'état (Trigger passe de OK à PROBLEM).  
  * Un **Problème** est un état (le Trigger *est* en état PROBLEM).


* **Action (Alerte) :** La réaction à un événement (ex: envoyer un email, exécuter un script).

* **Template (Modèle) :** Un ensemble préconfiguré d'Items, Triggers, Graphs, LLD, etc.

**Bonne Pratique Fondamentale :** Ne **JAMAIS**, au grand jamais, lier un Item ou un Trigger directement à un Hôte. **Toujours** créer/modifier un Template et lier ce Template à l'Hôte. Cela garantit la maintenabilité et la scalabilité.

#### **Module 4 : Rappels Protocoles (SNMP)**

Le protocole SNMP (Simple Network Management Protocol) est la "langue" universelle pour superviser les équipements réseau (switchs, routeurs, imprimantes, onduleurs).

* **Concepts Clés :**  
  * **Manager (Superviseur) :** Le **Serveur Zabbix**.  
  * **Agent (Supervisé) :** Le service SNMP sur le **Switch Cisco**.  
  * **Community String (Communauté) :** Un mot de passe (en clair) en SNMP v1 et v2c.  
  * **SNMPv3 :** La version moderne et sécurisée (authentification \+ chiffrement).


* **MIB et OID :**

  1. **MIB (Management Information Base) :** Le **dictionnaire** de l'équipement. Il décrit toutes les variables disponibles sous forme d'un **arbre hiérarchique**.  
  2. **OID (Object Identifier) :** L'**adresse unique** d'une variable dans l'arbre MIB.  
     * *Exemple :* .1.3.6.1.2.1.1.1.0 \= sysDescr (Description du système).  
     * *Exemple :* .1.3.6.1.2.1.2.2.1.8.x \= ifOperStatus (État opérationnel du port x).


![pix/z3.png](pix/z3.png)

![pix/z4.png](pix/z4.png)

* **Bonne Pratique (SNMP) :**  
  * Utiliser **SNMPv3** dès que possible.  
  * Si bloqué en v2c, utiliser une **communauté complexe** (pas public ou private).  
  * Utiliser des **ACLs** sur l'équipement réseau pour que *seul* le serveur Zabbix (ou son Proxy) ait le droit d'interroger en SNMP.

### 

### **Après-midi : Pratique \- Le Baptême du Feu**

#### **Atelier 1 : Installation de Zabbix 7.4 sur Ubuntu 24.04 (PostgreSQL \+ Nginx)**

**Objectif :** Installer un serveur Zabbix 7.4 fonctionnel avec sa base de données, son interface web et son agent local.

**1\. Prérequis Système:** 

Assurez-vous que votre serveur Ubuntu 24.04 est à jour.

`sudo apt update && sudo apt upgrade -y`  
`sudo timedatectl set-timezone 'Europe/Paris'`

**2\. Suivre les indications du Git \- Jour 1** 

[Atelier Complet Zabbix 7.4 — Ubuntu 24.04](https://github.com/spoofingcorp/zabbix_7.4/blob/main/Jour%201%20-%20Zabbix%207.4%20%E2%80%94%20Guide%20complet%20d%E2%80%99installation%20\(Serveur%20%2B%20Agent%20Linux%20%2B%20Dashboard\).md)  


**3\. Accès via l'Interface Web**

1. **Login :**  
   * **Username :** Admin (avec un 'A' majuscule)  
   * **Password :** zabbix


