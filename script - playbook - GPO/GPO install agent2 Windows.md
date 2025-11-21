# **Déploiement Agent2 Zabbix - Script PowerShell - GPO**

Si tu veux éviter le script PowerShell et utiliser la méthode "native" de déploiement logiciel Windows (**Installation Logicielle / Software Installation**), c'est possible, mais il y a une subtilité.

### Le problème de l'installation native
L'installateur MSI de Zabbix a besoin de paramètres (`SERVER`, `SERVERACTIVE`, etc.) pour fonctionner. Or, la console GPO classique "Installation de logiciel" **ne permet pas** de passer ces arguments directement (on ne peut pas juste taper `/qn SERVER=...`).

Pour faire du "100% GPO sans script", il faut utiliser un fichier de transformation **(.MST)**.

Voici la procédure "puriste" GPO (méthode MST) :

### Étape 1 : Créer le fichier de transformation (.MST)

Cette étape sert à "injecter" ta configuration (IP du serveur) directement dans le fichier d'installation MSI. Tu as besoin d'un petit outil gratuit de Microsoft appelé **Orca** (inclus dans le Windows SDK) ou un éditeur MSI équivalent.

1.  Installe et ouvre **Orca**.
2.  Ouvre le fichier `zabbix_agent2-7.4.5-windows-amd64-openssl.msi`.
3.  Dans le menu du haut, clique sur **Transform** > **New Transform**.
4.  Dans la colonne de gauche, cherche la table **Property**.
5.  Dans la partie droite, fais un clic droit > **Add Row** (Ajouter une ligne) pour ajouter tes configurations :
    * **Property:** `SERVER` | **Value:** `192.168.20.25`
    * **Property:** `SERVERACTIVE` | **Value:** `192.168.20.25`
    * **Property:** `HOSTNAME` | **Value:** (Laisse vide ou supprime la ligne si elle existe, pour qu'il prenne le nom de la machine Windows).
    * **Property:** `ENABLEPATH` | **Value:** `1`
6.  (Optionnel) Tu peux aussi chercher la table **ServiceInstall** pour vérifier que le service est bien en démarrage automatique.
7.  Une fois terminé, va dans **Transform** > **Generate Transform...**
8.  Sauvegarde le fichier sous `zabbix_config.mst` **dans le même dossier partagé** que ton fichier `.msi`.

### Étape 2 : Créer la GPO de déploiement

Maintenant que tu as le `.msi` (le programme) et le `.mst` (la config), on va dans l'AD.

1.  Ouvre la console de **Gestion de stratégie de groupe**.
2.  Crée une GPO "Deploy Zabbix Native".
3.  Va dans **Configuration ordinateur** > **Stratégies** > **Paramètres du logiciel** > **Installation de logiciel**.
4.  Clic droit > **Nouveau** > **Package...**
5.  Sélectionne ton fichier `zabbix_agent2...msi` (via le chemin réseau UNC `\\serveur\partage\msi`).
6.  **IMPORTANT :** Une fenêtre te demande la méthode de déploiement. Choisis **Avancé** (Advanced).
7.  La fenêtre de propriétés s'ouvre. Va dans l'onglet **Modifications**.
8.  Clique sur **Ajouter...** et sélectionne ton fichier `zabbix_config.mst`.
9.  Valide tout.

### Étape 3 : Le Pare-feu (Toujours nécessaire)

Même sans script, tu dois configurer le pare-feu via la GPO pour laisser passer les connexions, sinon le mode passif (commandes distantes) ne marchera pas.

1.  Dans la même GPO : **Configuration ordinateur** > **Paramètres Windows** > **Paramètres de sécurité** > **Pare-feu Windows...**
2.  Crée une **Règle de trafic entrant** :
    * Port : **10050** (TCP)
    * Action : Autoriser
    * Source (onglet Étendue) : `192.168.20.25`

### Résumé des différences

| Méthode | Avantages | Inconvénients |
| :--- | :--- | :--- |
| **Script de démarrage (PowerShell)** | Très flexible, facile à modifier (juste éditer le texte du script), logs faciles à créer. | Demande d'autoriser l'exécution de scripts PowerShell. |
| **GPO Native + MST (Orca)** | Méthode "propre" standard Microsoft, aucun script, tout est dans le package. | Modifier l'IP du serveur demande de refaire le fichier MST avec Orca. |

**Ma recommandation :** Si tu es à l'aise avec l'idée de générer un fichier `.mst` une fois, la méthode 2 est la plus propre pour un environnement Windows strict. Si tu penses devoir changer l'IP du serveur Zabbix souvent, la méthode par script est plus rapide à mettre à jour.