Documentatio Zabbix (Discord Webhook)**

### **Configuration côté Discord :**

1. Allez sur `https://discord.com/app` ou ouvrez l’application Desktop de Discord.
   Ouvrez les **Paramètres du serveur** puis allez dans l’onglet **Integrations**.

2. Cliquez sur **Create Webhook** pour créer un nouveau webhook.

3. Cliquez sur le webhook créé et modifiez ses informations si nécessaire.

4. Après avoir configuré votre webhook Discord, cliquez sur **Save Changes**.
   Vous pouvez maintenant copier l’URL du webhook en cliquant sur **Copy Webhook URL**, ou la consulter plus tard.

---

### **Configuration côté Zabbix :**

1. Avant d’utiliser le webhook Discord, configurez la macro globale `{$ZABBIX.URL}` :

   * Dans l'interface Zabbix, allez dans **Administration → Macros**.
   * Créez la macro globale **{$ZABBIX.URL}** contenant l’URL d’accès à votre interface Zabbix.
   * L’URL doit être une IP, un FQDN, ou localhost.
   * Le protocole (**http://** ou **https://**) est obligatoire.
   * Ajoutez éventuellement **/zabbix** selon la configuration de votre serveur Web.

   **Exemples corrects :**

   * [http://zabbix.com](http://zabbix.com)
   * [https://zabbix.lan/zabbix](https://zabbix.lan/zabbix)
   * [http://server.zabbix.lan/](http://server.zabbix.lan/)
   * [http://localhost](http://localhost)
   * [http://127.0.0.1:8080](http://127.0.0.1:8080)

   **Exemples incorrects :**

   * zabbix.com
   * [http://zabbix/](http://zabbix/)

2. Créez un utilisateur Zabbix et ajoutez-lui un média :

   * Allez dans **Users → Users**, cliquez sur **Create user**.
   * Dans l’onglet **User**, complétez les champs obligatoires.
   * Dans l’onglet **Media**, ajoutez un média et choisissez **Discord** dans la liste.
   * Dans **Send to**, collez l’URL du webhook Discord.
   * Vérifiez que cet utilisateur a accès aux hôtes pour lesquels vous voulez recevoir des notifications Discord.

3. Félicitations, vous pouvez maintenant utiliser ce média dans les actions Zabbix et recevoir des alertes sur votre serveur Discord !

La version la plus récente du script Discord est disponible ici :
[https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/media/discord](https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/media/discord)

---

# 🟣 **Procédure complète (claire et simplifiée) pour Zabbix 7.4**

## **1. Créer un webhook Discord**

Sur Discord :

➡️ **Serveur → Paramètres → Integrations → Webhooks**
➡️ **Create Webhook**
➡️ Donnez un nom (ex : `Zabbix Alerts`)
➡️ Copier l’URL générée
➡️ **Save**

---

## **2. Ajouter la macro globale dans Zabbix**

1. Zabbix → **Administration → Macros**
2. Cliquer **Create macro**
3. Remplir :

| Champ     | Valeur                               |
| --------- | ------------------------------------ |
| **Macro** | `{$ZABBIX.URL}`                      |
| **Value** | `https://votre_domaine_ou_ip/zabbix` |

> ⚠️ Important :
>
> * Si Zabbix est accessible via `http://192.168.1.50/`, mettez **[http://192.168.1.50/](http://192.168.1.50/)**
> * Si vous utilisez un reverse proxy, gardez l’URL publique.

---

## **3. Activer le média Discord dans Zabbix**

1. Aller dans **Administration → Media types**
2. Rechercher **Discord**
3. Vérifier que le média est **Enabled**
4. Aucune configuration supplémentaire n’est nécessaire ici (webhook JSON déjà préconfiguré par Zabbix)

---

## **4. Créer l’utilisateur qui recevra les alertes Discord**

1. **Administration → Users → Create user**
2. Renseigner :

   * Username : `discord-notifier`
   * Role : **User**
   * Autres champs obligatoires

---

## **5. Ajouter le média Discord à cet utilisateur**

1. Onglet **Media**
2. **Add**
3. Sélectionner **Media type : Discord**
4. Dans **Send to**, coller **l’URL du webhook Discord**
5. Définir les niveaux de sévérité :

Par exemple :

| Severity       | Statut  |
| -------------- | ------- |
| Not classified | Disable |
| Information    | Disable |
| Warning        | Enable  |
| Average        | Enable  |
| High           | Enable  |
| Disaster       | Enable  |

(Chez vous, on peut définir : pas de mails pour bas, mais Discord oui, etc.)

6. Enregistrer.

---

## **6. Configurer l'action Zabbix pour envoyer les alertes Discord**

1. Aller dans :
   **Configuration → Actions → Trigger actions**

2. Créer une nouvelle action :

   * Name : **Alerte Discord - Problèmes**
   * Condition :
     **Trigger severity ≥ Warning**
     (ou autre selon vos choix)

3. Onglet **Operations** :

   * **Add** → **Send message**
   * Sélectionner l’utilisateur `discord-notifier`
   * Sélectionner **Discord** comme média

4. Sauvegarder.

---

# 🟢 **Test immédiat**

Dans l'hôte → **Test → Test media**
Choisir l'utilisateur → média Discord.

Vous devez recevoir un message dans votre canal Discord.

---