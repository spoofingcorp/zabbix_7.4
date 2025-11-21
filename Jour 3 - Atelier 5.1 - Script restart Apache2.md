# 🛠️ Jour 3 - Atelier 5.1 - Auto-Remédiation (Self-Healing) Apache

**Scénario :** Zabbix surveille le service Apache sur Debian 13. Si le service s'arrête, Zabbix ordonne automatiquement à l'agent de le redémarrer.

-----

### Étape 1 : Préparation du Système (Sur Debian 13 ou Ubuntu 24.04)

- Prérequis, en hôte Linux avec l'agent 2 Zabbix installé + Apache2:

*Cette étape se déroule dans le terminal de votre serveur Debian (SSH).*

L'utilisateur `zabbix` doit avoir le droit de lancer la commande de redémarrage sans mot de passe.

**1. Configuration Sudo**
Créez un fichier de règles sudo spécifique :

```bash
sudo visudo /etc/sudoers.d/zabbix_remediation
```

Collez la ligne suivante (elle autorise uniquement le start/restart d'apache) :

```text
zabbix ALL=(root) NOPASSWD: /usr/bin/systemctl restart apache2, /usr/bin/systemctl start apache2
```

**2. Configuration de l'Agent Zabbix**
Par sécurité, l'exécution de commandes à distance est souvent bloquée par défaut.
Editez le fichier de configuration (souvent `/etc/zabbix/zabbix_agent2.conf`) :

```bash
sudo nano /etc/zabbix/zabbix_agent2.conf
```

Trouvez ou ajoutez la directive `AllowKey` pour autoriser l'exécution de commandes système :

```text
AllowKey=system.run[*]
```

Sauvegardez et redémarrez l'agent :

```bash
sudo systemctl restart zabbix-agent2
```

-----

### Étape 2 : Création du Script Global (Dans Zabbix UI)

*Nous définissons ici "l'outil" que Zabbix va utiliser.*

1.  Allez dans **Alerts** $\to$ **Scripts**.
2.  Cliquez sur le bouton **Create script** (en haut à droite).
3.  Remplissez le formulaire :
      * **Name :** `Restart Apache2`
      * **Scope :** Cochez **Action operations** (Indispensable pour l'automatisation).
      * **Type :** `Script`
      * **Execute on :** Sélectionnez `Zabbix agent` (Très important, sinon le serveur essaie de l'exécuter localement).
      * **Commands :**
        ```bash
        sudo systemctl restart apache2
        ```
      * **Timeout :** `10s` (Par défaut).
4.  Cliquez sur **Add**.

-----

### Étape 3 : La Détection (Item & Trigger)

*Nous définissons ici quand il y a un problème.*

**A. Créer l'Item (La surveillance)**

1.  Allez dans **Data collection** $\to$ **Hosts**.
2.  Cliquez sur **Items** à côté de votre hôte Debian.
3.  Cliquez sur **Create item** :
      * **Name :** `Apache Service Status`
      * **Type :** `Zabbix agent`
      * **Key :** `net.tcp.service[http]`
      * **Type of information :** `Numeric (unsigned)`
      * **Update interval :** `30s` (Un intervalle court permet une réparation plus rapide).
4.  Cliquez sur **Add**.

**B. Créer le Trigger (L'alerte)**

1.  Toujours dans la configuration de l'hôte, cliquez sur l'onglet **Triggers** (barre du haut).
2.  Cliquez sur **Create trigger** :
      * **Name :** `Apache service is DOWN on {HOST.NAME}`
      * **Severity :** `High`
      * **Expression :** (Vous pouvez la taper ou utiliser le constructeur) :
        ```text
        last(/NomDeVotrHote/net.tcp.service[http])=0
        ```
        *(Remplacez `NomDeVotrHote` par le nom technique exact de l'hôte dans Zabbix)*.
3.  Cliquez sur **Add**.

-----

### Étape 4 : L'Automatisation (Action)

*C'est le lien entre le Trigger (problème) et le Script (solution).*

1.  Allez dans **Alerts** $\to$ **Actions** $\to$ **Trigger actions**.
2.  Cliquez sur **Create action**.
3.  **Onglet "Action" :**
      * **Name :** `Auto-Heal Apache2`
      * **Conditions :**
          * Cliquez sur **Add**.
          * Type : `Trigger`.
          * Operator : `equals`.
          * Value : Sélectionnez votre trigger `Apache service is DOWN...`.
4.  **Onglet "Operations" :**
      * Repérez la section **Operations** (la première zone).
      * Cliquez sur **Add**.
      * **Operation type :** `Global script`.
      * **Script name :** Sélectionnez `Restart Apache2` (créé à l'étape 2).
      * **Target list :** Cochez **Current host**.
      * Cliquez sur le petit bouton **Add** (dans la popup).
5.  Cliquez sur le bouton final **Add** (en bas de page) pour sauvegarder.

-----

### Étape 5 : Test Grandeur Nature

1.  **Côté Serveur Debian (SSH)** : Arrêtez le service web.
    ```bash
    sudo systemctl stop apache2
    ```
2.  **Côté Zabbix UI** :
      * Allez dans **Monitoring** $\to$ **Problems**.
      * Attendez environ 30 secondes (votre intervalle).
      * Le problème apparaît. Regardez la colonne **Actions**. Vous devriez voir le nom de l'action ou un statut "Executed".
3.  **Vérification Finale (SSH)** :
    ```bash
    systemctl status apache2
    ```
    Le service doit être `Active: active (running)`.


_______(Optionnel)_Ajout de la Notification de Réussite (Recovery Operation)_______

Confirmer que le script a bien fonctionné et que le service est retombé sur ses pattes (le Trigger est passé de l'état "Problem" à "Resolved"). 

Voici comment ajouter cette notification dans l'action que nous avons créée précédemment.

-----

### 🛠️ Ajout de la Notification de Réussite (Recovery Operation)

Nous repartons de l'action existante.

1.  Allez dans **Alerts** $\to$ **Actions** $\to$ **Trigger actions**.
2.  Cliquez sur le nom de votre action : `Auto-Heal Apache2`.
3.  Allez dans l'onglet **Operations** (en haut du formulaire).
4.  Descendez jusqu'à la section nommée **Recovery operations** (c'est en dessous de la section "Operations" où nous avions mis le script).
5.  Cliquez sur **Add**.

### Configuration de la notification

Dans la fenêtre pop-up qui s'ouvre :

  * **Operation type :** Sélectionnez `Send message`.
  * **Send to user groups :** (Optionnel) Sélectionnez `Zabbix administrators`.
  * **Send to users :** Sélectionnez votre utilisateur (ex: `Admin` ou `Guest` selon votre config).
  * **Default media type :** Vous pouvez laisser sur `- All -` ou forcer `Email` si vous l'avez configuré.
  * **Custom message :** Cochez cette case si vous voulez un texte personnalisé (plus clair).
      * **Subject :** `Auto-Heal Success: {EVENT.NAME}`
      * **Message :**
        ```text
        Le service Apache a été redémarré avec succès sur {HOST.NAME}.

        Durée de la panne : {EVENT.DURATION}
        Action effectuée : Redémarrage automatique par l'agent Zabbix.
        État actuel : OK
        ```
  * Cliquez sur le petit bouton **Add** (dans la pop-up).

### Validation finale

6.  De retour sur la page principale de l'action, cliquez sur le bouton **Update** (en bas de page).

-----

### 💡 Comment tester ?

1.  Refaites le test d'arrêt (`sudo systemctl stop apache2`).
2.  Attendez que Zabbix détecte la panne (Problème) et lance le script.
3.  Dès que le script a redémarré Apache, l'item `net.tcp.service[http]` va repasser à `1` (Up).
4.  Le Trigger va passer à l'état **Resolved**.
5.  Zabbix va exécuter la **Recovery operation** et vous devriez recevoir l'email (ou voir l'envoi dans **Reports** $\to$ **Action log**).


