
# 🛠️ Jour 2 - Atelier 4 - Optimisation Alerting - Zabbix 7.4

## 1\. Canaux de Communication (Media Types)

L'objectif est d'utiliser les connecteurs natifs Zabbix 7 (plus fiables que les simples requêtes HTTP).

### A. Configurer Discord

**Chemin :** `Alerts` → `Media types` → `Discord`

1.  Si absent, importer le modèle officiel via le bouton "Import".
2.  **Configuration du script** : Laisser le script JS par défaut.
3.  **Parameters** :
      * `discord_endpoint` : Mettre `{ALERT.SENDTO}`
      * *Pourquoi ?* Cela permet de définir l'URL du Webhook directement dans la fiche de chaque utilisateur (plus flexible).

### B. Configurer MS Teams

**Chemin :** `Alerts` → `Media types` → `MS Teams`

1.  Utiliser le template natif.
2.  **Parameters** :
      * `teams_endpoint` : `{ALERT.SENDTO}` (Même logique que Discord).

-----

## 2\. Routage des Alertes (Users)

**Chemin :** `Users` → `Users` → *Sélectionner l'Admin ou l'équipe*

Dans l'onglet **Media**, ajouter chaque canal :

| Type | Send to (La destination) | When active | Severity |
| :--- | :--- | :--- | :--- |
| **Email** | `admin@societe.com` | 1-7,00:00-24:00 | Cocher tout |
| **Discord** | `https://discord.com/api/webhooks/YOUR_ID/YOUR_TOKEN` | 1-7,00:00-24:00 | High, Disaster |
| **MS Teams** | `https://your-teams-webhook-url` | 1-7,00:00-24:00 | High, Disaster |

-----

## 3\. Cerveau des Alertes (Actions & Escalades)

C'est ici que nous configurons le message initial, la répétition (toutes les 15 min) et la résolution.

**Chemin :** `Alerts` → `Actions` → `Trigger actions` → Créer une action "Report Problems to Admin"

### Onglet : Operations (Le début et la répétition)

*Ici, nous définissons que le message part immédiatement, puis se répète indéfiniment tant que le problème dure.*

  * **Default operation step duration** : `15m` (Le rythme)

**Step 1 : Notification Initiale + Répétition**

  * **Steps** : `1` to `0` (0 signifie "Infini")
  * **Send to users** : Admin
  * **Subject** : `PROBLEM: {EVENT.NAME}`
  * **Message** :
    ```text
    🔴 PROBLEM DETECTED
    Host: {HOST.NAME}
    Issue: {EVENT.NAME}
    Severity: {EVENT.SEVERITY}
    Current Value: {ITEM.LASTVALUE}

    Duration: {EVENT.AGE}
    (Ce message se répètera toutes les 15 minutes)
    ```

> **Note importante :** En mettant "Steps 1 to 0", Zabbix envoie le premier mail tout de suite, puis attend 15 minutes (la durée par défaut) et renvoie le même message, et ainsi de suite.

### Onglet : Recovery Operations (La fin)

*Message unique quand tout rentre dans l'ordre.*

  * **Notify all involved** : `Coché`
  * **Subject** : `✅ RESOLVED: {EVENT.NAME}`
  * **Message** :
    ```text
    ✔ ISSUE RESOLVED
    Host: {HOST.NAME}
    Issue: {EVENT.NAME}

    Duration: {EVENT.DURATION}
    Recovery Time: {EVENT.RECOVERY.TIME}
    ```

-----

## 4\. Capteurs Intelligents (Triggers - Syntaxe Z7)

Voici les versions corrigées et optimisées de vos triggers pour éviter les faux positifs.

### A. Linux – Espace Disque (Avec Hystérésis réelle)

*On alerte à \< 5%, mais on ne coupe l'alerte que si ça remonte à \> 10% pour éviter l'effet "clignotant".*

  * **Name** : `Low disk space on {HOST.NAME} (Volume: {ITEM.VALUE1})`
  * **Problem Expression** (Déclencheur) :
    ```javascript
    last(/Linux Server/vfs.fs.size[/,pfree]) < 5
    ```
  * **Recovery Expression** (Fermeture) :
    ```javascript
    min(/Linux Server/vfs.fs.size[/,pfree],10m) > 10
    ```

### B. Cisco – Interface DOWN (Filtrée)

*On alerte seulement si le port est DOWN techniquement ALORS qu'il devrait être UP administrativement.*

  * **Name** : `Interface Gi1/0/1 is DOWN (Unexpected)`
  * **Expression** :
    ```javascript
    last(/Switch Cisco/net.if.status[ifOperStatus.10101]) = 2
    and
    last(/Switch Cisco/net.if.admin_status[ifAdminStatus.10101]) = 1
    ```

### C. Cisco – Taux d'erreur (Corrigé)

*Correction : Utilisation de la fonction `rate` car les erreurs sont des compteurs incrémentaux.*

  * **Name** : `High error rate on interface Gi1/0/1 (> 5 err/sec)`
  * **Expression** :
    ```javascript
    rate(/Switch Cisco/net.if.in.errors[ifInErrors.10101], 5m) > 5
    ```
    *(Signifie : Plus de 5 erreurs par seconde en moyenne sur les 5 dernières minutes).*

-----

## 5\. Anti-Bruit (Dépendances)

Pour empêcher Zabbix de spammer quand un équipement central tombe.

**Concept :** Si le Switch Core est HS, il est inutile de dire que les serveurs derrière sont injoignables.

**Configuration :**

1.  Aller sur la liste des Triggers du **Serveur Linux**.
2.  Trouver le trigger **"Zabbix agent is unreachable"** (ou "ICMP Ping Fail").
3.  Cliquer dessus → Onglet **Dependencies**.
4.  Cliquer sur **Add**.
5.  Sélectionner le **Switch Cisco** → Choisir le trigger **"ICMP Ping"** (ou "System is unavailable").

## **Tests réels**

## **6.1 Test Linux – Disque plein**

Sur le serveur Linux :

```bash
fallocate -l 10G /tmp/fill_disk
```

Puis :

* Monitoring → Latest data → vfs.fs.size
* Trigger → PROBLEM
* Alerte → Test escalade + recovery

---

## **6.2 Test Cisco – Port DOWN**

Switch :

```
conf t
int gi1/0/1
shutdown
```

Vérifier :

* Alerte High → Email + Discord
* Recovery à l’activation
