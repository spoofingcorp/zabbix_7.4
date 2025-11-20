# Jour 2 - Atelier 4 - Optimisation Alerting & Triggers (Compatible Zabbix 7.4)

## 1\. Configuration des Notifications (Actions)

Nous allons configurer des messages de résolution clairs et des rappels périodiques tant que le problème persiste.

### 1.1 Message de Résolution (Recovery Operations)

*Permet de clôturer proprement le ticket ou d’alerter les équipes que l'incident est fini.*

  * **Chemin :** Alerts → Actions → Trigger actions.
  * Choisir l'action concernée (ou en créer une).
  * Onglet **Recovery operations**.
  * **Operation** : `Notify all involved` (ou Send message).
  * **Sujet :** `RESOLVED: {EVENT.NAME} on {HOST.NAME}`
  * **Message :**
    ```text
    The issue "{EVENT.NAME}" on {HOST.NAME} has been resolved.
    Resolved at: {EVENT.RECOVERY.TIME} on {EVENT.RECOVERY.DATE}
    Duration: {EVENT.DURATION}
    Original ID: {EVENT.ID}
    ```

### 1.2 Rappels Périodiques (Escalations)

*Pour rappeler toutes les 15 minutes que le problème est toujours actif. Attention : Dans Zabbix, cela se configure dans l'onglet "Operations" via les étapes (Steps), et non dans "Update operations".*

  * Onglet **Operations**.
  * **Default operation step duration** : `15m` (C'est ici qu'on définit l'intervalle).
  * **Steps** :
      * Cliquer sur "Edit" (ou Add).
      * **From** : `1`
      * **To** : `0` (Signifie "Infini" : répéter tant que le problème est là).
  * **Message** :
    ```text
    ⚠ ALERTE EN COURS
    The issue {EVENT.NAME} is still active on {HOST.NAME}.
    Elapsed time: {EVENT.AGE}
    Last value: {ITEM.LASTVALUE}
    Severity: {EVENT.SEVERITY}

    Please take action immediately.
    ```

-----

## 2\. Configurer Discord & Teams (Méthode Native Zabbix 7)

Zabbix 7 utilise des scripts JavaScript natifs pour les Webhooks.

### 2.1 Création du Media Type

**Chemin :** Alerts → Media types.

**Pour Discord :**

1.  Chercher **Discord**. S'il n'existe pas, importer le fichier YAML/XML officiel ou créer un type `Webhook`.
2.  **Parameters** :
      * Vérifier la présence de `discord_endpoint`.
      * Valeur : `{ALERT.SENDTO}` (Cela permet de définir l'URL Webhook unique pour chaque utilisateur).
      * *Alternative :* Coller l'URL du Webhook Discord directement ici si c'est pour tout le monde pareil.

**Pour MS Teams :**

1.  Chercher **MS Teams**.
2.  Attention : Utilisez le script compatible "Workflows" (car les connecteurs Office 365 sont dépréciés).
3.  **Parameters** : `teams_endpoint` → `{ALERT.SENDTO}`.

### 2.2 Associer le média à l'utilisateur

**Chemin :** Users → Users.

1.  Cliquer sur votre utilisateur (ex: `Admin`).
2.  Onglet **Media** → **Add**.
3.  **Type :** Discord.
4.  **Send to :** *Coller ici l'URL complète de votre Webhook Discord*.
5.  **Type :** MS Teams.
6.  **Send to :** *Coller ici l'URL du Workflow Teams*.
7.  **Severity :** Cocher les cases souhaitées (souvent : Warning, Average, High, Disaster).

-----

## 3\. 🔔 Configuration des Triggers (Syntaxe Zabbix 7.4)

### 3.1 Linux – Disque faible (Trigger Intelligent)

L'agent Zabbix 2 utilise désormais des "Dependent Items". La clé `vfs.fs.size` est souvent remplacée par `vfs.fs.dependent.size`.
Nous allons utiliser le pourcentage utilisé (`pused`) avec une logique d'hystérésis pour éviter le "flapping" (alerte qui clignote).

**Scénario :** Alerter si le disque est plein à \> 95% pendant 5 minutes. Rétablir quand il redescend sous 90%.

  * **Name :** Low disk space on {HOST.NAME} (Used: {ITEM.LASTVALUE})
  * **Severity :** High

**Expression (Problem) – Le déclencheur :**

```javascript
min(/Linux Server/vfs.fs.dependent.size[/,pused], 5m) > 95
```

> *Traduction :* Si l'espace utilisé est resté **constamment** au-dessus de 95% durant les 5 dernières minutes.

**Recovery Expression – La fermeture (OK) :**

```javascript
max(/Linux Server/vfs.fs.dependent.size[/,pused], 2m) < 90
```

> *Traduction :* L'alerte ne s'arrête que si l'espace utilisé redescend (au maximum) sous les 90% pendant 2 minutes.

### 3.2 Cisco Switch – Triggers SNMP

#### 3.2.1 Interface DOWN

Alerte immédiate si un port important tombe.

  * **Name :** Interface Gi1/0/1 changed state to DOWN
  * **Severity :** High
  * **Expression :**
    ```javascript
    last(/Switch Cisco/net.if.status[ifOperStatus.10101]) = 2
    ```
    *(Optionnel : Ajouter `and last(/Switch Cisco/net.if.admin_status[...]) = 1` pour ignorer les ports éteints volontairement).*

#### 3.2.2 Taux d’erreur élevé (Correction du TP initial)

Ne jamais utiliser `min` ou `last` brut sur un compteur d'erreurs (`ifInErrors`), car ce chiffre ne fait qu'augmenter. Il faut mesurer la **vitesse** d'augmentation (le débit d'erreurs).

  * **Name :** High error rate on interface Gi1/0/1
  * **Severity :** Warning
  * **Expression :**
    ```javascript
    rate(/Switch Cisco/net.if.in.errors[ifInErrors.10101], 10m) > 2
    ```
    > *Traduction :* Si on détecte plus de 2 erreurs par seconde en moyenne sur 10 minutes.

### 3.3 Dépendances — Anti-bruit massif

Cas : Si le switch tombe, on ne veut pas recevoir 50 alertes "Serveur injoignable".

1.  Aller sur le trigger du **Serveur Linux** (ex: "Zabbix Agent unreachable" ou "ICMP Ping").
2.  Onglet **Dependencies** → **Add**.
3.  Sélectionner le **Switch Cisco** → Trigger **"ICMP Ping Loss"** (ou Status Down).

✔ **Résultat :**

  * Si le switch coupe ➡️ **1 seule alerte** (Switch Down).
  * Les alertes des serveurs connectés sont mises en pause (supprimées) automatiquement.

## 4\. 🧪 Tester et Valider la Configuration

Il ne faut jamais attendre une vraie panne pour vérifier si l'alerte part bien sur Discord ou Teams. Voici comment simuler les pannes correspondant aux triggers créés plus haut.

### 4.1 Tester l'alerte Disque (Hystérésis)

Puisque nous avons configuré un délai de déclenchement de **5 minutes** (`min(..., 5m)`), le test demande un peu de patience.

**Sur le serveur Linux surveillé :**

1.  **Remplir le disque artificiellement :**
    Créons un fichier de 10 Go (ajuster la taille selon la taille de votre disque pour dépasser les 95%).
    ```bash
    # Créer un fichier 'dummy' rempli de zéros
    dd if=/dev/zero of=/tmp/disk_fill_test bs=1G count=10
    ```
2.  **Observer dans Zabbix :**
      * Allez dans *Monitoring → Hosts → Latest data*.
      * Regardez la valeur `Used space percentage` monter.
      * **Attendre 5 minutes** (la condition `min` doit être vraie pendant toute la durée).
3.  **Vérification :**
      * Le trigger doit passer en état "PROBLEM".
      * Vous devez recevoir la notification Discord/Teams.
      * Zabbix doit afficher le message d'escalade après 15 min si non résolu.
4.  **Résolution :**
    ```bash
    rm /tmp/disk_fill_test
    ```
      * L'alerte doit se fermer (Status OK) et envoyer le message de "Recovery".

### 4.2 Tester la dépendance (Switch vs Serveur)

Pour vérifier que vous ne recevez pas d'alerte serveur quand le switch tombe.

1.  **Action :** Couper l'accès réseau du Switch (ou simuler une panne SNMP).
      * *Astuce simple :* Changer temporairement l'adresse IP du switch dans la configuration de l'hôte Zabbix (ex: mettre une IP bidon).
2.  **Résultat attendu :**
      * L'alerte **Switch Down** (ICMP Ping) apparaît.
      * Attendre 2-3 minutes.
      * Vérifier que l'alerte **Zabbix Agent Unreachable** du serveur Linux n'apparaît **PAS** (ou apparaît grisée/supprimée dans le dashboard).

-----

## 5\. 🏷️ Organisation par Tags (Bonnes pratiques Zabbix 7)

Dans Zabbix 7, les **Tags** remplacent les anciennes catégories. Ils permettent de filtrer quelles alertes vont vers Discord (DevOps) et quelles alertes vont vers Teams (Management).

### 5.1 Ajouter des Tags au Trigger

Reprenez votre Trigger "Low disk space".

  * Onglet **Tags**.
  * Ajouter :
      * **Name :** `Scope` | **Value :** `OS`
      * **Name :** `Team`  | **Value :** `SysAdmin`
      * **Name :** `Impact`| **Value :** `Critical`

### 5.2 Filtrer l'envoi dans l'Action

Pour éviter de spammer tout le monde, on modifie l'action d'envoi (Action Log).

1.  Allez dans **Alerts → Actions → Trigger actions**.
2.  Ouvrez votre action (ex: "Report problems to Discord").
3.  Onglet **Action** (Conditions).
4.  Ajouter une condition :
      * `Tag value` : **Team** = `SysAdmin`
5.  Sauvegarder.

✔ **Résultat :** Seules les alertes taguées "SysAdmin" partiront vers ce canal Discord. Une alerte taguée "Database" pourrait partir vers un autre canal.

-----

## 6\. 🛠️ Gestion des Maintenances

Pour éviter de recevoir des alertes pendant une mise à jour planifiée (ex: reboot d'un serveur).

**Chemin :** Data collection → Maintenance.

1.  **Create maintenance period**.
2.  **Name :** Mise à jour Hebdomadaire.
3.  **Maintenance type :**
      * *With data collection* (On continue de voir les graphiques, mais pas d'alertes).
      * *No data collection* (Aveugle complet).
4.  **Periods :**
      * One time only (si ponctuel).
      * Weekly (ex: Tous les dimanches à 3h du matin).
5.  **Hosts and Groups :** Sélectionner le groupe "Linux Servers".

✔ **Effet :** Pendant cette période, les triggers peuvent passer en "Problème" dans l'interface, mais **aucune Action (Notification)** ne sera déclenchée. L'icône de maintenance (clé orange) apparaîtra sur le dashboard.

-----

## 📝 Résumé de l'atelier

| Fonctionnalité | Avant (Zabbix \<6) | Maintenant (Zabbix 7.4) |
| :--- | :--- | :--- |
| **Disque** | `vfs.fs.size` direct | `vfs.fs.dependent.size` (Master items) |
| **Discord/Teams** | Scripts Bash/Python externes | Webhooks natifs JavaScript |
| **Anti-Flapping** | Trigger hysteresis complexe | Syntaxe `min/max` simplifiée dans l'expression |
| **Filtrage** | Groupes d'hôtes | Tags (Etiquettes) |

