# Jour 3 – Atelier 7

# **Reporting PDF Planifié (Scheduled Reports) – Zabbix 7.4**

Objectif : Générer automatiquement un **rapport PDF** contenant un dashboard Zabbix et l’envoyer par e-mail chaque semaine (ex. : chaque lundi à 08h00).

Cette fonctionnalité est native et disponible depuis Zabbix 6+.

---

# 1. Prérequis indispensables

Avant de créer un rapport planifié, vérifier les éléments suivants :

## 1.1 SMTP configuré

**Chemin :** *Administration → Media types → Email*

* SMTP host : serveur SMTP
* SMTP port
* Authentification activée
* From address configurée

→ Un test d’envoi doit être fonctionnel.

---

## 1.2 Frontend Zabbix accessible par une URL HTTPS

**Obligatoire** car le moteur d’export PDF capture l’interface Zabbix via un navigateur Headless (Chromium).

Vérifier dans **Administration → General → GUI → Frontend URL** :

* **Frontend URL :**
  `https://monitoring.example.com/zabbix/`

---

## 1.3 Permissions des utilisateurs

Les utilisateurs recevant le rapport doivent avoir accès :

* au **dashboard**,
* aux **données** qu’il contient.

Sinon : PDF vide ou erreur d’export.

---

# 2. Création du Reporting Planifié

**Chemin :** *Monitoring → Scheduled reports* → **Create scheduled report**

Renseigner les champs comme suit.

---

## 2.1 Informations générales

| Paramètre        | Valeur                                                                 |
| ---------------- | ---------------------------------------------------------------------- |
| **Name**         | `📝 Rapport Hebdomadaire – Cockpit`                                    |
| **Dashboard**    | `🔴 Cockpit – Supervision Live`                                        |
| **Format**       | `PDF`                                                                  |
| **Report width** | `1920` (idéal pour un rendu TV ou Full HD)                             |
| **Description**  | `Rapport hebdomadaire incluant problèmes, charge et état des disques.` |

---

## 2.2 Programme d’envoi (Schedule)

**Type :** `Weekly`
**Day of week :** `Monday`
**Time :** `08:00`
**Timezone :** configurer le fuseau approprié (Europe/Paris dans ton cas).

Le rapport sera généré et envoyé automatiquement chaque lundi matin.

---

## 2.3 Destinataires

Cliquer sur **Add** dans “Users / User groups”.

Exemples :

* User : `admin`
* User : `noc-team`
* User group : `SysAdmin`

Chaque destinataire utilise le média configuré dans son profil (ex. e-mail).

---

## 2.4 Options avancées

### Wait for data synchronization

> À activer si vous utilisez un **proxy Zabbix** pour garantir que les données soient à jour avant génération du PDF.

### Retry attempts

> Définir 1 à 3 tentatives en cas d’échec d’envoi SMTP.

---

# 3. Vérification et Test

Après création du rapport :

1. Aller dans : *Monitoring → Scheduled reports*
2. Cliquer sur **…** → **Test**
3. Le PDF doit arriver dans votre boîte mail dans les secondes qui suivent.

En cas de problème :

* vérifier le **frontend URL**,
* vérifier les **logs Apache / Nginx**,
* vérifier les **logs du serveur SMTP**.

---

# 4. Structure du PDF généré

Le PDF contient :

* l’intégralité du dashboard sélectionné,
* tous les widgets visibles à l’écran,
* l’heure de génération,
* votre logo personnalisé (si configuré dans Administration → Branding).

Le rendu correspond exactement à l’affichage TV en mode **Kiosk**.

---

# 5. Bonnes Pratiques

### 5.1 Un dashboard dédié pour les rapports

Éviter les dashboards interactifs contenant des filtres dynamiques.
Créer un **dashboard statique** spécialement pour les PDF.

### 5.2 Privilégier une largeur de 1920 px

Meilleur rendu pour les widgets Honeycomb, graphes, Top hosts.

### 5.3 Limiter les widgets trop larges

Le PDF n’aime pas les graphes horizontaux trop longs.

### 5.4 Vérifier les permissions

Si un widget affiche “No permission”, il sera vide dans le PDF.

---

# 6. Résultat final

Une fois configuré :

* **Chaque lundi à 8h**, Zabbix génère automatiquement un PDF.
* Le document inclut tous les widgets critiques du `Cockpit – Supervision Live`.
* Le rapport est envoyé par e-mail à votre équipe de supervision.

Cela constitue un excellent support NOC pour démarrer la semaine.


