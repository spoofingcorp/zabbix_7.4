# 🏷️ Atelier 9 - Zabbix Tagging : Guide de l'Architecte Senior (v5.0-7.x)

(voir également https://www.initmax.cz/wp-content/uploads/2023/11/power_of_tags_70_en.pdf)

Le **tagging** est la fonctionnalité la plus sous-estimée mais la plus puissante de Zabbix moderne (depuis la 5.0, et encore plus critique en 7.x). Si vous commencez à implémenter Zabbix dans votre SI, ce guide vous aidera à le structurer comme un architecte Zabbix Senior.

-----

## 1\. 💡 Philosophie : Le Tag comme "Clé de Routage"

Dans Zabbix 7.x, il faut **cesser de penser "Hostgroups" pour les alertes**. Il faut penser **"Event Metadata"**.

  * **Avant** : On alertait parce que le serveur était dans le groupe `"Linux Servers"`.
  * **Maintenant** : On alerte parce que l'incident porte l'étiquette `Team: SysAdmin` et `Env: Production`.

### La Matrice de Tagging (Taxonomie Standard)

Pour que cela fonctionne, il faut imposer une convention de nommage stricte. Voici la **"Sainte Trinité"** des tags pour le routage d'alertes :

| Nom du Tag | Description | Exemple de Valeurs | Niveau d'application |
| :--- | :--- | :--- | :--- |
| **Target** (ou **Scope**) | Quel est l'objet technique touché ? | `OS`, `Oracle`, `Docker`, `Nginx` | **Template** (Général) |
| **Team** | Qui doit recevoir le ticket/mail ? | `Ops`, `Dev_Backend`, `DBA`, `Secu` | **Trigger** ou **Hôte** |
| **Env** | Quel est l'environnement ? | `Prod`, `PreProd`, `Dev`, `DR` | **Hôte** (Indispensable) |
| **App** | Quel service métier est impacté ? | `E-Commerce`, `SAP`, `Intranet` | **Hôte** |

-----

## 2\. 🏗️ Implémentation : Où placer quel tag ?

L'erreur classique est de tout mettre au niveau de l'Hôte. C'est faux. L'héritage des tags est **cumulatif** dans Zabbix (`Hôte` + `Template` + `Trigger`).

### A. Au niveau du Template (Tags Statiques)

À utiliser pour classer la **technologie**.

```markdown
Template : OS Linux
Tag : Target: Linux
```

> **Utilité** : Permet de filtrer dans la vue "Dernières données" tout ce qui concerne Linux, quel que soit le serveur.

### B. Au niveau de l'Hôte (Tags d'Infrastructure)

À utiliser pour le **contexte géographique et logique** (environnement, application métier, datacenter).

```markdown
Hôte : srv-prod-db-01
Tag : Env: Production
Tag : Datacenter: PAR-DC2
Tag : App: Billing
```

> **Résultat** : Ces tags seront collés sur tous les problèmes de ce serveur. Si le CPU explose, l'alerte dira : "CPU High sur Billing (Prod)".

### C. Au niveau du Déclencheur/Trigger (Tags de Routage - LE PLUS CRITIQUE)

C'est ici que se joue l'**intelligence de routage**. Dans un même Template (ex: Oracle), certains triggers concernent les SysAdmins, d'autres les DBAs.

| Trigger | Description | Tag à ajouter |
| :--- | :--- | :--- |
| `Service Oracle Down` | Problème de service systemd | `Team: SysAdmin` |
| `Tablespace Full` | Problème d'espace de données Oracle | `Team: DBA` |

#### Astuce Zabbix 7 (Macros dans les Tags)

Utilisez les **macros de découverte bas niveau (LLD)** dans les tags des prototypes de triggers pour ajouter du contexte dynamique.

```markdown
Trigger Prototype : Espace disque faible sur {#DEVNAME}.
Tag : Component: Disk_{#DEVNAME}
```

> **Résultat** : L'alerte sortira avec le tag `Component: Disk_sda1` ou `Component: Disk_data`, ce qui est très utile pour le debug ou les tickets.

-----

## 3\. ⚙️ Configuration des Actions (Le moteur de notification)

C'est ici que la magie opère. Au lieu de créer une action par Hostgroup, vous créez une **action par Équipe**.

#### Action : "Notification Team DBA - PROD"

**Conditions :**

  * `Tag value` : `Team` **égal à** `DBA`
  * `Tag value` : `Env` **égal à** `Production`
  * `Severity` : **est supérieur ou égal à** `Moyen`

**Opérations :**

  * `Envoyer vers Media Type` : `PagerDuty_DBA` ou `Channel_Slack_DBA`.

> **Pourquoi est-ce génial ?** Si demain vous ajoutez un serveur PostgreSQL, vous n'avez pas à modifier les actions. Vous mettez juste le tag `Team: DBA` dans le template PostgreSQL, et le routage se fait tout seul.

-----

## 4\. ✅ Retours d'expérience (REX) & Bonnes Pratiques Zabbix 7

### ✅ Ce qu'il faut faire

  * **Utiliser les Tags pour les Maintenances** :

      * Filtrez vos périodes de maintenance sur les tags au lieu des ID d'hôtes.
      * *Exemple* : Maintenance "Patching Linux Mensuel" -\> Filtre sur **Tags** `Target: Linux` **ET** `Env: PreProd`.

  * **Corrélation d'événements globale** :

      * Zabbix permet de fermer automatiquement des problèmes basés sur des tags.
      * *Exemple* : Si un trigger `"Lien Fibre Down"` (Tag `Scope: Network`) se déclenche, on peut masquer automatiquement toutes les alertes `"Agent inatteignable"` qui portent le même Tag `Datacenter`.

  * **Les Tableaux de Bord (Widgets)** :

      * Les widgets "Problèmes par sévérité" de Zabbix 7 sont puissants avec les tags. Créez un Dashboard "Vue DBA" qui ne filtre que les problèmes avec le tag `Team: DBA`.

### ❌ Les pièges à éviter (Anti-patterns)

  * **La "Cardinalité" Explosive** :

      * **Ne mettez jamais de valeurs uniques aléatoires** dans les noms de tags.
      * **Mauvais** : `Tag ErrorID: 4923842` (Cela va tuer la base de données Zabbix si vous avez des millions d'événements).
      * **Bon** : Gardez des valeurs finies (catégories) comme `ErrorType: DB_Connection`.

  * **L'inconsistance de la casse** :

      * Zabbix est **sensible à la casse**. `env: prod` est différent de `Env: Prod`.
      * **Solution** : Forcez le **PascalCase** (`PremièreLettreMajuscule`) pour tout le monde dans la charte de nommage.

  * **Tagger les Items (Éléments)** :

      * Tagger les items est possible, mais **rarement utile pour l'alerting** (les alertes viennent des triggers). Ne surchargez pas vos items de tags, sauf si vous utilisez massivement la vue "Dernières données" pour des filtres précis.

-----

## 5\. 🛠️ Proposition d'Exercice Pratique

### Scénario : Le Routage Intelligent

L'objectif est de prouver que le même serveur peut alerter des équipes différentes en fonction du problème technique.

1.  **Tagging des Triggers (Template Linux)** :

      * Identifier le Trigger **"High CPU load"**. Lui ajouter le tag `Team: Ops`.
      * Identifier le Trigger **"/etc/passwd has been changed"**. Lui ajouter le tag `Team: Secu`.

2.  **Création des Actions d'Alerte** :

      * Créer une Action 1 : **Si Tag** `Team` **=** `Ops` -\> Envoi Email Admin.
      * Créer une Action 2 : **Si Tag** `Team` **=** `Secu` -\> Envoi SMS Responsable Sécurité.

3.  **Test et Validation** :

      * Lancer un `stress -c 4` (High CPU Load) -\> **Vérifier que seul l'Admin reçoit l'alerte**.
      * Faire un `touch /etc/passwd` -\> **Vérifier que seul la Sécurité reçoit l'alerte**.

