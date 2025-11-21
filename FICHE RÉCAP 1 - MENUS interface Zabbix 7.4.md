# ✅ **1. FICHE RÉCAP – MENUS interface Zabbix 7.4**

---

# 🟥 DASHBOARDS

### **Rôle : Vue d’ensemble et analyse rapide de l’état du SI**

Le Dashboard est l’écran principal pour le suivi global.
On y retrouve des widgets permettant de visualiser :

* charge CPU
* disponibilité hôtes
* problèmes actifs
* graphes de performance
* SLA
* dernière collecte SNMP/agent
* état des proxys

**Utilisation concrète :**

* écran NOC (rafraîchissement automatique)
* vue consolidée pour direction / équipe supervision
* vérification instantanée de l'état global

---

# 🟦 MONITORING

Section essentielle pour l’analyse opérationnelle en temps réel.

---

## **1. Problems**

Vue centrale des incidents actifs (triggers = PROBLEM).
Affiche :

* heure de détection
* hôte concerné
* sévérité
* description du trigger
* actions liées (notifications, scripts…)
* état acknowledgment

**Usage réel :**
Analyse des incidents prioritaires du SI.

---

## **2. Hosts**

Synthèse par hôte :

* disponibilité ICMP
* disponibilité agent
* disponibilité SNMP
* état général

**Usage réel :**
Vérifier rapidement si un hôte n’est plus joignable.

---

## **3. Latest data**

Liste complète des **items** collectés par hôte :

* dernière valeur
* horodatage
* mini/max/avg
* graphes instantanés

**Usage réel :**
Diagnostic rapide : “Est-ce que la donnée remonte ?”

---

## **4. Maps**

Cartographies dynamiques :

* liens colorés selon triggers
* états des hôtes
* représentation réseau

**Usage réel :**
Monitoring graphique des architectures réseau / datacenter.

---

## **5. Discovery**

Affiche les résultats des scans réseau L3/L2 :

* hôtes trouvés
* services détectés
* SNMP trouvé / non trouvé
* actions automatiques appliquées

**Usage réel :**
Découverte automatique d’un LAN, ajout automatisé des switches/VM/PC.

---

# 🟩 SERVICES

### **Rôle : Supervision orientée disponibilité / SLA**

Permet de construire une vue “métier” :

Exemple :

```
Service Web
 ├─ Nginx
 ├─ API
 └─ Base de données
```

Chaque élément peut dépendre d’autres triggers.
Zabbix calcule ensuite :

* disponibilité (%)
* durée des interruptions
* impact métier

**Usage réel :**
Rapports SLA pour la DSI ou les clients (infogérance).

---

# 🟪 INVENTORY

### **Rôle : Base CMDB légère intégrée à Zabbix**

Contient :

* numéro de série
* modèle
* emplacement
* OS
* version firmware
* propriétaire du matériel

**Usage réel :**
Tracer les équipements facilement sans outil externe.

---

# 🟧 REPORTS

### **Rôle : Statistiques, audits et rapports automatisés**

---

## **1. System information**

Vue technique Zabbix :

* version
* modules chargés
* performance moteur
* threads
* pollers

## **2. Scheduled Reports**

Envoi automatique de rapports PDF.
Exemples :

* rapport quotidien d’incidents
* disponibilité hebdomadaire

## **3. Availability report**

Disponibilité par hôte ou groupe, basé sur les pings/triggers.

## **4. Top 100 triggers**

Analyse de pollution d’alertes.
Très utile pour :

* réduire les faux positifs
* optimiser les templates

## **5. Audit log**

Historique des actions d’administration.

## **6. Notifications**

Historique des alertes envoyées (mail, webhook, scripts).

---

# 🟥 DATA COLLECTION

La zone la plus importante : **toute la configuration de la supervision**.

---

## **1. Templates**

C’est le cœur de la standardisation.
Un template contient :

* items
* triggers
* macros
* graphes
* LLD (discovery automatique)

**Usage réel :**
Créer une règle unique pour 200 serveurs identiques.

---

## **2. Host groups**

Classification logique des hôtes.

Exemples :

* Linux
* Windows
* VMware
* Cisco
* Datacenter-Paris
* Production / Préprod / Test

---

## **3. Hosts**

Configuration unitaire d’un équipement supervisé :

* interfaces (agent, SNMP, JMX, IPMI)
* macros
* templates appliqués
* items spécifiques
* triggers personnalisés
* dashboard spécifique à l’hôte

---

## **4. Maintenance**

Définit des plages sans alertes :

* patch OS
* mises à jour
* migration VM
* arrêt planifié

Zabbix continue de collecter mais **ne crée pas d’alertes**.

---

## **5. Event correlation**

Très puissant pour réduction de bruit :

* supprime les alertes enfant si l’alerte parent existe
* supprime les doublons
* corrige automatiquement des enchaînements d’événements

---

## **6. Discovery**

Configuration des “Network discovery rules” :

* scan ICMP
* scan SNMP avec version + communauté
* port scanner
* exécution d’actions automatiques (ex. : ajouter un switch)

---

# 🟥 ALERTS

---

## **1. Actions**

Automatisation complète :

* notifications mail
* notifications Teams/Discord
* escalades
* exécution de scripts (bash/powershell)
* conditions avancées

## **2. Media types**

Connecteurs pour envoyer des alertes :

* SMTP
* Slack
* Discord
* Microsoft Teams
* Webhook HTTP
* Telegram
* SMS

## **3. Scripts**

Commandes exécutées depuis Zabbix vers un hôte :
Exemples :

* redémarrer un service
* vérifier un port
* nettoyer un cache
* exécuter un script de diagnostic

---

# 🟫 USERS

---

## **User groups**

Utilisé pour attribuer des droits par groupe (NOC, Admins, Réseau…).

## **Roles**

RBAC avancé :

* accès aux menus
* accès aux hôtes
* droits lecture/écriture

## **Users**

Comptes d’accès (MFA activable).

---

# 🟪 ADMINISTRATION

---

## **General**

Configuration globale Zabbix :

* fuseaux horaires
* affichage
* paramètres de supervision globale
* encryption
* messages systemd

## **Housekeeping**

Nettoyage automatique :

* historique
* tendances
* événements

## **Proxy groups / Proxies**

Configuration du maillage Zabbix Proxy.
Utile pour :

* sites distants
* segmentation réseau
* haute charge

## **Macros**

Macros globales appliquées partout.

## **Queue**

Liste des items en retard de collecte.
Outil de diagnostic critique.

---

