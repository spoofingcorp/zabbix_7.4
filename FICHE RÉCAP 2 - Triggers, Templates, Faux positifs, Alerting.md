# ✅ **2. FICHE RÉCAP – Notions fondamentales Zabbix 7.4**


# Fiche récapitulative Zabbix (Jour 1 et 2)

## Templates et Modifications
Les templates permettent de définir des **items**, **triggers**, **graphes**, **LLD** et autres éléments communs à plusieurs hôtes.  
Les modifications s’effectuent directement dans l’interface de Zabbix.  
Une fois un template ajusté, il est **automatiquement appliqué** à tous les hôtes liés, évitant la configuration manuelle et améliorant la standardisation.

---

## Triggers et Problèmes
Un **trigger** est une condition logique qui, lorsqu’elle devient vraie, génère un **problème**.  
Les problèmes peuvent être **acknowledged** pour indiquer qu’ils sont pris en charge.  
Ils se ferment automatiquement lorsque la condition du trigger redevient normale.

---

## Hystérésis (Faux Positifs)
L’hystérésis permet d’empêcher un trigger de changer trop fréquemment d’état lorsque la valeur surveillée oscille autour du seuil.  
Il réduit ainsi les **faux positifs** et évite le phénomène de “flapping”.

---

## Escalade des Problèmes
Le mécanisme d’escalade permet de transmettre une alerte à un niveau supérieur si le problème n’est pas résolu dans un délai défini.  
Il assure une montée en intensité du traitement en fonction de l’urgence ou de la durée d’inactivité.

---

## Dépendances de Trigger
Un trigger peut dépendre d’un autre afin d’éviter les alertes multiples pour un même incident.  
Exemple :  
Si un hôte est entièrement hors ligne, les triggers individuels (CPU, disque, interfaces…) ne doivent pas se déclencher de manière indépendante.

---

## Actions de Trigger
Les triggers peuvent effectuer plusieurs types d’actions :
- Envoi d’emails
- Notifications webhook (Teams, Slack, Discord…)
- Exécution de scripts distants
- Commandes automatisées (correctives ou d’investigation)

Ces actions permettent d’améliorer la réactivité et l’automatisation dans la gestion des incidents.

# ✅ **3. FICHE RÉCAP – Notions fondamentales Zabbix 7.4**

---

# 🟦 Host

Objet principal surveillé :

* serveur
* VM
* routeur
* switch
* service cloud

L’hôte contient :

* items
* triggers
* macros
* interfaces
* low-level discovery
* dashboard dédié

---

# 🟩 Item

Une métrique collectée.
Exemples :

* `system.cpu.load`
* `vfs.fs.size[/,pfree]`
* `net.if.in[eth0]`
* SNMP OID `.1.3.6.1.2.1.2.2…`

Types :

* agent
* SNMP
* HTTP
* calculated
* dependent
* trapper
* Zabbix agent active
* script

---

# 🟥 Trigger

Détection d’état anormal.
Exemples :

* **Disque faible**
  `last(/server/vfs.fs.size[/,pfree]) < 10`

* **Interface down**
  `min(/Cisco/net.if.in[1],5m) = 0`

Un trigger = une alerte potentielle.

---

# 🟨 Problem

Généré lorsqu’un trigger passe en PROBLEM.
Apparaît dans :

* Monitoring → Problems
* Dashboard
* Actions (notification)

---

# 🟧 Graph

Graphique généré automatiquement.
Apparaît dans Latest Data > Graph ou Dashboard.

---

# 🟪 LLD (Low Level Discovery)

Détection automatique :

* interfaces réseau
* filesystems
* disques
* OSPF neighbors
* VLANs (Cisco)
* BGP peers
* VM VMware

Essentiel pour switchs, VMware, serveurs multi-NIC.

---

# 🟫 Network Discovery

Balayage réseau par plages IP.
Détecte :

* pings
* agents
* SNMP
* ports ouverts

Peut créer automatiquement les hôtes.

---

# 🟩 Dashboard

Personnalisable avec widgets :

* top CPU
* problèmes actifs
* disponibilité
* graphes
* SLA
* carte réseau
* widget texte HTML

---

# 🟦 DAUT (Auto Update Timer)

Rafraîchissement automatique du dashboard.
Paramétrable entre 5 s et plusieurs minutes.

---

# ✅ **4. FICHE RÉCAP — Triggers, Templates, Faux positifs, Alerting**

---

# 🟥 Templates

Contiennent :

* items
* triggers
* graphes
* macros
* LLD
  → Base de configuration centrale.

Toujours modifier un template plutôt qu’un hôte.

---

# 🟦 Triggers

Détection logique :

* seuils
* changement d’état
* comparaison dans le temps
* dépendances

---

# 🟧 Dépendances

Permet d’éviter les alertes cascades :
Exemple :

* Host down
  → Cache les alertes CPU/RAM/Service

---

# 🟫 Hystérésis – Anti-flapping

Exemple :

```
max(/metabase2023/vfs.fs.dependent.size[/,pused],#5)>50
```

→ Pour forcer la continuité du problème durant 5 minutes :


---

# 🟪 Escalations

Notifier si le problème n’est pas corrigé après X minutes.

---

# 🟩 Actions

Automatisations :

* email
* Teams
* Discord
* script shell
* redémarrage service
* isolation réseau

---

# ✅ **2. SYNTHÈSE TRIGGER – Nouvelle syntaxe**


### ✔ **Ancienne logique**

Déclenche l’alerte lorsque *l’espace libre* < X %

```
last(/linux/vfs.fs.size[/,pfree]) < 10
```

### ✔ **Nouvelle logique moderne (Zabbix 7.x)**

Déclenche l’alerte lorsque *l’espace utilisé* reste > X % pendant une certaine durée.

### **Trigger conforme Zabbix 7.4 :**

```
max(/metabase2023/vfs.fs.dependent.size[/,pused],#5) > 50
```

### ✔ Explications pédagogiques :

| Élément                 | Rôle                                         |
| ----------------------- | -------------------------------------------- |
| `/metabase2023/`        | Hôte supervisé                               |
| `vfs.fs.dependent.size` | Item dépendant (pused)                       |
| `[/,pused]`             | Partition "/" – pourcentage utilisé          |
| `max(...,#5)`           | Prend les **5 dernières valeurs**            |
| `> 50`                  | Condition : dépassement de 50% d’utilisation |

### ✔ Avantage : anti-flapping intégré

Car il faut 5 collectes successives au-dessus du seuil.

### ✔ Résultat

Trigger fiable, idéal en production.

---



