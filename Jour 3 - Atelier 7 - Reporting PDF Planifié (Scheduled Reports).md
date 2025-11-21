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
  `https://monitoring.example.com/ # Ou l'IP du Serveur Zabbix`


La fonctionnalité de rapports PDF (Scheduled Reports) introduite depuis Zabbix 5.4/6.0 nécessite une architecture particulière qui n'est pas activée par défaut.

Voici les étapes pour corriger ce problème sur votre Zabbix (la procédure est identique pour la version 7.x) :

### 1\. Comprendre l'architecture

Pour générer un PDF, Zabbix Server ne le fait pas tout seul. Il a besoin de deux composants supplémentaires :

1.  **Zabbix Web Service :** Un service autonome qui fait le pont.
2.  **Google Chrome (ou Chromium) :** Utilisé en mode "headless" pour faire le rendu visuel du dashboard avant de le convertir en PDF.

-----

### 2\. Installation des paquets requis

Si ce n'est pas déjà fait, vous devez installer le service web et le navigateur.

**Sur RHEL/CentOS/AlmaLinux/Rocky :**

```bash
dnf install zabbix-web-service google-chrome-stable
```

*(Note : Si `google-chrome-stable` n'est pas trouvé, vous pouvez utiliser `chromium`)*.

**Sur Debian/Ubuntu :**

```bash
apt install zabbix-web-service chromium
```

### 3\. Configuration du Serveur Zabbix

C'est l'étape cruciale qui cause votre message d'erreur. Vous devez dire au serveur Zabbix de démarrer les processus "Report Writers".

1.  Ouvrez le fichier de configuration : `/etc/zabbix/zabbix_server.conf`
2.  Recherchez et modifiez (ou ajoutez) les lignes suivantes :

<!-- end list -->

```ini
# Ce paramètre doit être au moins à 1 (par défaut il est à 0, ce qui cause votre erreur)
StartReportWriters=1

# L'URL où écoute le zabbix-web-service (par défaut port 10053)
WebServiceURL=http://localhost:10053/report
```

### 4\. Configuration du Zabbix Web Service

Vérifiez le fichier `/etc/zabbix/zabbix_web_service.conf`. Généralement, la configuration par défaut suffit, mais assurez-vous que le port correspond (10053).

Ensuite, activez et démarrez le service :

```bash
systemctl enable --now zabbix-web-service
```

### 5\. Redémarrage du Serveur Zabbix

Pour prendre en compte le changement `StartReportWriters`, redémarrez le serveur :

```bash
systemctl restart zabbix-server
```

-----

### 6\. Configuration de l'URL Frontend (Interface Web)

Dernière étape : Le service web (Chrome) doit savoir à quelle adresse URL se connecter pour prendre la "photo" du dashboard.

1.  Allez dans l'interface web Zabbix.
2.  Menu **Administration** -\> **General** -\> **Other** (ou **Autre**).
3.  Cherchez le champ **Frontend URL**.
4.  Mettez l'URL complète de votre interface Zabbix (ex: `http://192.168.1.50/`).
      * *Note : Cette URL doit être accessible depuis le serveur lui-même.*

### Résumé du dépannage

Si l'erreur persiste après ces étapes, vérifiez les logs du serveur (`/var/log/zabbix/zabbix_server.log`). Si vous voyez une erreur parlant de "handshake" ou de "connection refused", c'est souvent que le `zabbix-web-service` ne tourne pas ou que le port 10053 est bloqué par un pare-feu local.

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


