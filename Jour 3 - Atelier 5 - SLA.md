# Jour 3 - Atelier 5 : Ingénierie SLA pour Services Web (Zabbix 7.4)

**Objectif :** Construire une chaîne de surveillance complète qui part d'une vérification technique (ping HTTP) pour aboutir à un rapport de disponibilité managérial (SLA 99.9%), en utilisant les meilleures pratiques de templating.


---

### Étape 0 : Comprendre le concept
Dans ce scénario, Zabbix (le serveur ou le proxy) va agir comme un client qui navigue sur votre site. L'hôte que nous créons sert juste de **boîte** pour ranger le scénario web et les données SLA.

### Étape 1 : Création de l'Hôte
1.  Allez dans **Collecte de données → Hôtes**.
2.  Cliquez sur le bouton **Créer un hôte** (en haut à droite).

### Étape 2 : Configuration de l'Hôte (Onglet Hôte)
Remplissez les champs comme suit :

* **Nom de l'hôte :** `Srv_Web_Portail_Client`
    * *Conseil : Donnez-lui un nom technique unique sans espaces.*
* **Nom visible :** `Portail Client (Web)`
    * *Conseil : C'est ce nom qui s'affichera sur les cartes et les graphiques.*
* **Modèles (Templates) :**
    * C'est ici que vous liez le travail fait précédemment.
    * Tapez ou sélectionnez : `Template App HTTPS SLA Gold` (celui que nous avons importé/créé).
* **Groupes d'hôtes :**
    * Sélectionnez ou créez un groupe, par exemple : `Applications Web` ou `Services Critiques`.
* **Interfaces :**
    * *Subtilité :* Pour un scénario web pur, l'interface n'est pas strictement utilisée (car l'URL est dans le scénario), mais Zabbix en demande souvent une par défaut ou pour faire un "Ping" simple.
    * Cliquez sur **Ajouter** → **Agent**.
    * **Adresse IP :** `127.0.0.1` (Si vous voulez juste que l'hôte existe) **OU** mettez l'IP réelle de votre site web (`192.x.x.x`) si vous souhaitez aussi surveiller la latence réseau (Ping) vers ce serveur.
    * **Port :** `10050` (Laissez par défaut).

### Étape 3 : Configuration des Macros (Votre point de départ)
C'est maintenant que vous appliquez la consigne de la Phase 1.

1.  Restez dans la fenêtre de configuration de l'hôte.
2.  Cliquez sur l'onglet **Macros** (en haut).
3.  Cliquez sur **Macros héritées et du modèle** (Bouton important).
    * *Pourquoi ?* Parce que vous avez lié le modèle à l'étape 2, les macros `{$WEB.URL}`, etc. devraient déjà apparaître en "gris" (héritées).
4.  Cliquez sur **Modifier** (le petit lien à côté de la valeur héritée) ou ressaisissez simplement la macro pour la surcharger spécifiquement pour cet hôte.
    * Macro : `{$WEB.URL}` ⇒ Valeur : `https://votre-vrai-site.com`

### Étape 4 : Validation
Cliquez sur le bouton **Ajouter** (en bas).

---

### Résumé : À quoi ressemble votre hôte ?
Vous avez maintenant un hôte `Srv_Web_Portail_Client` qui :
1.  Contient le modèle SLA.
2.  Possède les bonnes URLs via les Macros.
3.  Ne nécessite **aucun agent Zabbix installé** sur le serveur web cible (c'est le serveur Zabbix qui fait le travail à distance).

Une fois cet hôte créé, le scénario web (Phase 2 de l'atelier) se lancera automatiquement sous 1 minute.


---

## Phase 2 : La Sonde (Scénario Web)

Nous créons le robot qui va tester le site.

**Emplacement :** Dans le Modèle ou l'Hôte, section **Scénarios Web**.
**Bouton :** Créer un scénario web.

### Onglet 1 : Scénario
* **Nom :** `Check.Web.Portail`
* **Intervalle de mise à jour :** `1m` (Indispensable pour un calcul SLA précis).
* **Tentatives :** `2` (Pour absorber les micro-coupures réseau sans fausser le SLA).
* **Agent :** Sélectionnez `Chrome` ou `Firefox` (Pour passer les pare-feux basiques).

### Onglet 2 : Étapes (Steps)
C'est ici que vous définissez la navigation.
* **Étape 1 : Accueil**
    * **URL :** `{$WEB.URL}` (Utilisation de la Macro définie en Phase 1).
    * **Codes d'état requis :** `200`.
    * **Chaîne requise :** `Copyright` (OPTIONNEL - NON OBLIGATOIRE - Ou un texte unique du footer pour valider le chargement complet).

*(Cliquez sur Ajouter pour sauvegarder le scénario)*

---

## Phase 3 : La Détection et le Marquage (Trigger)

Le SLA a besoin d'un signal clair pour savoir quand le service est "en panne".

**Emplacement :** Onglet **Déclencheurs** (Triggers).
**Bouton :** Créer un déclencheur.

1.  **Nom :** Le service Web {$WEB.URL} est indisponible.
2.  **Expression :** `last(/NomHote/web.test.fail[Check.Web.Portail])<>0`
    * *Traduction : Si le numéro de l'étape échouée est différent de 0, c'est une panne.*
3.  **Sévérité :** `Haute` ou `Désastre`.
4.  **TAGS (Crucial pour le SLA) :**
    * Allez dans l'onglet **Tags**.
    * Nom : `ServiceTarget`
    * Valeur : `App_Portail_Web`
    * *Note : Ce tag sert de "balise" pour que le Service Métier repère l'incident.*

---

## Phase 4 : La Couche Service (Services Métier)

Nous quittons la technique pour le métier.

**Emplacement :** Menu **Services → Services**.
**Action :** Cliquez sur **Éditer** (en haut à droite) puis **Créer un service**.

1.  **Onglet Service :**
    * **Nom :** Portail Client (Vue Métier).
    * **Algorithme :** Le plus critique des services fils.
2.  **Onglet Tags (Identité du Service) :**
    * Ajoutez un tag pour classifier ce service dans une gamme de SLA.
    * Nom : `SLA_Tier`
    * Valeur : `Gold`
3.  **Onglet Tags de problème (Liaison Technique) :**
    * C'est ici qu'on "écoute" le déclencheur de la Phase 3.
    * Nom : `ServiceTarget`
    * Opérateur : `Égal`
    * Valeur : `App_Portail_Web`

*(Le service est maintenant vivant : si le trigger sonne, le service passe au rouge).*

---

## Phase 5 : Le Contrat (SLA)

Définition des règles du jeu (Objectifs et Calculs).

**Emplacement :** Menu **Services → SLA**.
**Bouton :** Créer un SLA.

1.  **Nom :** Contrat SLA Gold (99.9%).
2.  **SLO :** `99.9`.
3.  **Période :** `Mensuelle`.
4.  **Fuseau horaire :** (Ex: Europe/Paris).
5.  **Tags de service (Le périmètre) :**
    * Quels services sont concernés par ce contrat ?
    * Nom : `SLA_Tier`
    * Opérateur : `Égal`
    * Valeur : `Gold`
    * *Magie : Tous les services créés en Phase 4 avec le tag "SLA_Tier: Gold" seront automatiquement calculés par ce SLA.*

---

# Phase 6 - Crash Test

Pour tester que votre mécanique de SLA fonctionne sans réellement casser votre serveur de production, la méthode la plus sûre et la plus efficace consiste à **tromper Zabbix** en modifiant la configuration de surveillance.

Voici 3 méthodes pour simuler un crash, de la plus douce à la plus radicale.

### Méthode 1 : Le Sabotage de la Macro (Recommandée)
C'est la méthode la plus propre car elle teste toute la chaîne (Macro -> Scénario -> Trigger -> Service) sans toucher au serveur réel.

1.  Allez dans **Collecte de données → Hôtes**.
2.  Cliquez sur votre hôte, onglet **Macros**.
3.  Modifiez la macro `{$WEB.URL}`.
    * *Valeur actuelle :* `https://www.mon-service.com`
    * *Nouvelle valeur :* `https://www.mon-service.com/cette-page-n-existe-pas-12345` (ou une URL invalide).
4.  Cliquez sur **Mettre à jour**.
5.  **Attendez 2 minutes** (Le temps que le cache de configuration Zabbix se rafraîchisse et que le scénario s'exécute).

### Méthode 2 : La Simulation de "Page Blanche" (Test de contenu)
Ce test est excellent pour vérifier si Zabbix détecte une page qui charge (Code 200) mais qui est vide ou en erreur applicative.

1.  Allez dans le **Scénario Web**, onglet **Étapes**.
2.  Ouvrez l'étape "1. Accueil".
3.  Modifiez le champ **Chaîne requise** (Required string).
    * *Valeur actuelle :* `Copyright`
    * *Nouvelle valeur :* `UnePhraseQuiNExistePasSurLeSite` (ex: "BananaSplit").
4.  Sauvegardez.
5.  Zabbix va se connecter, recevoir un code 200, mais échouera car il ne trouve pas le mot "BananaSplit". Il considérera le site comme HS.

### Méthode 3 : L'Arrêt Brutal (Si vous avez accès au serveur)
Si vous surveillez un environnement de test, coupez simplement le service web.

* En ligne de commande sur le serveur cible : `systemctl stop nginx` (ou apache2).

---

### Comment vérifier la réaction en chaîne ?

Une fois la panne simulée, ne regardez pas seulement le trigger. Vérifiez la cascade complète pour valider votre architecture SLA :



1.  **Niveau Sonde (Technique) :**
    * Allez dans **Surveillance → Hôtes → Web**.
    * Le scénario doit être rouge. L'erreur affichée sera explicite (ex: *Error: 404* ou *Required pattern not found*).

2.  **Niveau Alerte (Déclencheur) :**
    * Sur votre tableau de bord, le problème **"Le service Web... est indisponible"** doit apparaître.
    * **Point clé :** Cliquez sur le nom du problème pour voir les détails. Vérifiez que le Tag `ServiceTarget: App_Portail_Web` est bien présent sur l'événement.

3.  **Niveau Métier (Service) :**
    * Allez dans **Services → Services**.
    * Votre service `Portail Client` doit être passé automatiquement à l'état **Problème (Rouge)**.
    * *Si le service reste vert alors que le trigger sonne, c'est que votre Tag de liaison est mal configuré.*

4.  **Niveau Contrat (SLA) :**
    * Allez dans **Services → Rapport SLA**.
    * Générez le rapport sur "Aujourd'hui".
    * Vous verrez que le **SLI** (Barre verte) a légèrement baissé (ex: 99,92%) et que le **Budget d'Erreur** a été consommé.

### Pour revenir à la normale
1.  Remettez la bonne URL dans la Macro `{$WEB.URL}` (ou la bonne chaîne de caractères).
2.  Attendez 1 minute.
3.  Le trigger va se résoudre (passer au vert/disparaître).
4.  Le Service va repasser à l'état OK.
5.  Le temps de panne sera enregistré à jamais dans le rapport SLA du mois.


## Phase 7 : Visualisation (Tableau de Bord)

Rendre la donnée visible.

1.  Allez dans **Tableaux de bord**.
2.  Passez en mode édition et ajoutez un widget.
3.  Type : **Rapport SLA**.
4.  **SLA :** Sélectionnez `Contrat SLA Gold`.
5.  **Service :** Sélectionnez `Portail Client`.
6.  **Période :** `Mois en cours`.

---

### Récapitulatif du flux de données

| Niveau | Objet | Action |
| :--- | :--- | :--- |
| **Hôte** | Macro `{$WEB.URL}` | Définit l'adresse cible. |
| **Collecte** | Scénario Web | Teste l'URL chaque minute. |
| **Alerte** | Déclencheur | Passe en alerte si échec + **Pose le Tag `ServiceTarget`**. |
| **Métier** | Service | **Lit le Tag `ServiceTarget`** pour changer son état. |
| **Contrat** | SLA | Calcule le temps "OK" vs "KO" du Service. |






