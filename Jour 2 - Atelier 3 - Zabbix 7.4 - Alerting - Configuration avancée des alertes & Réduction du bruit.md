# **🧪 Jour 2 - Atelier 3 - Zabbix 7.4 \- Alerting Configuration avancée des alertes & Réduction du bruit**

**Objectif :** Configurer une chaîne d'alerte complète et robuste (Mail, Discord, Teams) avec une logique d'escalade et de filtrage des faux positifs sur un environnement Zabbix 7.4 moderne.  
**Périmètre :**

* **Agent Zabbix (Linux) :** Alerte d'espace disque intelligent.  
* **Switch Cisco (SNMP) :** Détection de coupure port et erreurs.  
* **Canaux :** E-mail, Discord, Teams.  
* **Stratégie :** Warning (Chat uniquement) vs High/Disaster (Mail \+ Chat).

## **1\. 📌 Pré-requis & Navigation Zabbix 7**

### **1.1 Hôte Linux (Agent2)**

* **Chemin :** Data collection (Collecte de données) → Hosts (Hôtes).  
* **Vérification :** L'hôte doit être "Enabled" et l'interface **ZBX** doit être verte.  
* **Template associé :** *Linux by Zabbix agent* (ou *agent 2*).

### **1.2 Switch Cisco (SNMPv2)**

* **Chemin :** Data collection → Hosts.  
* **Template :** *Cisco IOS by SNMP* (Standard moderne) ou *Network Generic Device by SNMP*.  
* **Vérification :** L'interface **SNMP** doit être verte.

## **2\. 📧 Configuration des Médias (Alerts)**

**Note :** Dans Zabbix 7, la configuration des médias se trouve désormais sous le menu "Alerts".

### **2.1 Configurer le serveur SMTP (E-mail)**

1. Aller dans : Alerts → Media types.  
2. Cliquer sur **Email** (ou le cloner pour en créer un dédié).  
3. **Paramètres :**  
   * **SMTP server :** smtp.exemple.com  
   * **Port :** 587 (ou 25/465 selon fournisseur)  
   * **Connection security :** STARTTLS (recommandé)  
   * **Authentication :** Username and password  
   * **Message format :** HTML  
4. **Onglet Message templates :** Vérifier que les templates par défaut (*Problem*, *Recovery*) sont présents.  
5. **Test :** Cliquer sur "Test" à droite de la ligne Email → Saisir votre mail → **Send**.

### **2.2 Configurer Discord & Teams (Méthode native Zabbix 7\)**

Zabbix 7 possède des connecteurs natifs plus puissants que les webhooks bruts.  
**Pour Discord :**

* Alerts → Media types.  
* Chercher **Discord**. S'il n'est pas là : Import (fichier XML officiel Zabbix) ou créer un nouveau type "Webhook".  
* *Si Webhook manuel simple :*  
  * **Type :** Webhook  
  * **Parameters :** Ajouter discord\_endpoint avec l'URL de votre Webhook Discord.

**Pour Teams :**

* Alerts → Media types → **MS Teams**.  
* Remplir le paramètre teams\_endpoint avec l'URL de votre connecteur Teams.

### **2.3 Associer les médias à l'utilisateur**

1. Aller dans : Users (Utilisateurs) → Users.  
2. Cliquer sur votre utilisateur (ex: Admin).  
3. Onglet **Media** → **Add**.  
   * **Type :** Email → Renseigner votre adresse.  
   * **Type :** Discord → Renseigner l'URL ou l'ID.  
   * **Type :** MS Teams → Renseigner l'URL.  
4. **Severity :** Cocher **toutes les cases** pour le moment (le filtrage se fera via les Actions).

## **3\. 🔔 Configuration des Triggers (Nouvelle Syntaxe Z7)**

**Attention :** La syntaxe {Host:key.last()} est obsolète. Nous utilisons last(/Host/key).

### **3.1 Agent Linux — Disque faible (Trigger intelligent)**

* **Chemin :** Data collection → Hosts → Triggers (Linux Server).  
* **Action :** Create trigger.

**Exemple : Espace disque critique (\< 5%)**

* **Name :** Low disk space on {HOST.NAME} (Volume: {ITEM.VALUE1})  
* **Severity :** High  
* **Expression (Nouvelle syntaxe) :**  
  `last(/Linux Server/vfs.fs.size[/,pfree]) < 5`

#### **3.1.1 Réduction des faux positifs (Hysteresis)**

Pour éviter le "flapping" (oscillation), on demande que la valeur reste basse pendant 5 minutes.

* **Expression modifiée :**  
  `max(/Linux Server/vfs.fs.size[/,pfree],5m) < 5`  
  *(Traduction : Si le maximum d'espace libre sur les 5 dernières minutes est inférieur à 5%, alors alerte. Cela signifie que pendant 5 minutes, l'espace n'est jamais repassé au-dessus de 5%).*

### **3.2 Switch Cisco — Port Down & Erreurs**

#### **3.2.1 Trigger : Port Down (Opérationnel)**

* **Name :** Interface Gi1/0/1 changed state to DOWN  
* **Severity :** High  
* **Expression :**  
  `last(/Switch Cisco/net.if.status[ifOperStatus.10101]) = 2`  
  *(Note : 10101 est l'index SNMP de l'interface, à adapter selon votre découverte).*

#### **3.2.2 Trigger : Taux d'erreur élevé (Maintenance prédictive)**

* **Name :** High error rate on interface Gi1/0/1  
* **Severity :** Warning  
* **Expression :**  
  `min(/Switch Cisco/net.if.in.errors[ifInErrors.10101],10m) > 5`  
  *(Alerte seulement si on a plus de 5 erreurs en continu pendant 10 minutes).*

## **4\. 📬 Actions (Le cerveau des notifications)**

**Chemin :** Alerts → Actions → Trigger actions.

### **4.1 Action A : "Incidents Critiques" (Mail \+ Chat)**

1. **Create action**.  
2. **Name :** Critical Alerting (High/Disaster).  
3. **Conditions :**  
   * Severity is equals to **High**  
   * Severity is equals to **Disaster**  
4. **Operations :**  
   * Send message to users : **Admin**.  
   * Send only to : **Email, Discord, MS Teams**.

### **4.2 Action B : "Incidents Mineurs" (Chat uniquement)**

1. **Create action**.  
2. **Name :** Warning Alerting (Chat Only).  
3. **Conditions :**  
   * Severity is equals to **Warning**  
   * *(Optionnel)* Severity is equals to **Average**  
4. **Operations :**  
   * Send message to users : **Admin**.  
   * Send only to : **Discord** (et/ou Teams).  
   * 🔴 **Important :** Ne pas sélectionner Email ici.

## **5\. 🛠️ Best Practices Zabbix 7 (Anti-bruit)**

### **5.1 Recovery Expression (Anti-flapping)**

Pour éviter qu'un service qui oscille (UP/DOWN toutes les 10s) ne spamme :

1. Dans le Trigger, basculer **Problem generation mode** sur **Recovery expression**.  
2. **Problem Expression :** last(/Host/cpu.util) \> 95 (Seuil haut)  
3. **Recovery Expression :** last(/Host/cpu.util) \< 80 (Seuil bas) *Résultat : L'alerte se déclenche à 95%, mais ne se ferme que si le CPU redescend sous 80%.*

### **5.2 Dépendances (Trigger Dependencies)**

**Cas classique :** Si le routeur est DOWN, tous les serveurs derrière sont injoignables.

1. Ouvrir le trigger **"Zabbix Agent unreachable"** de votre serveur Linux.  
2. Onglet **Dependencies**.  
3. Ajouter une dépendance vers le trigger **"ICMP Ping Down"** de votre Switch/Routeur. *Résultat : Si le switch tombe, Zabbix n'envoie qu'une seule alerte (le switch) et masque celles des serveurs.*

## **6\. 📊 Vérification & Test**

### **6.1 Simulation Disque**

Sur le serveur Linux :  
`# Créer un fichier de 10Go (ajuster selon la taille disque)`  
`fallocate -l 10G /tmp/fill_disk`

1. Aller dans Monitoring → Hosts.  
2. Cliquer sur **Latest data** pour le Linux.  
3. Attendre ou forcer **Execute now** sur l'item vfs.fs.size.  
4. Vérifier Monitoring → Problems.

### **6.2 Simulation Port**

Sur le Switch Cisco (si accès) :  
`conf t`  
`int gi1/0/1`  
`shutdown`

Vérifier que l'alerte arrive bien par **Mail ET Discord** (car High severity).

## **📦 Résumé des acquis**

À la fin de cet atelier Zabbix 7.4, vous maîtrisez :

* ✅ La nouvelle navigation (Data collection / Alerts).  
* ✅ La nouvelle syntaxe de triggers (last(/host/key)).  
* ✅ La ségrégation des alertes (Grave \= Réveil, Warning \= Log).  
* ✅ L'hysteresis pour stabiliser la surveillance.