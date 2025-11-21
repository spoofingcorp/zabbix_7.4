# Ansible Playbook - Install Agent2 Windows - mode active

Puisque les MSI Zabbix acceptent des arguments d'installation (pour remplir le fichier de conf automatiquement), la méthode la plus robuste n'est pas d'utiliser la section "Installation logicielle" de la GPO (trop rigide), mais d'utiliser un **script de démarrage (Startup Script)** via GPO.

Voici le processus étape par étape pour déployer l'agent Zabbix 7.4 en **mode Actif**.

### 1\. Préparation des sources (Le Partage Réseau)

Il faut que tes serveurs puissent accéder au fichier `.msi`.

1.  Télécharge l'installateur MSI depuis ton lien : [zabbix\_agent2-7.4.5-windows-amd64-openssl.msi](https://www.google.com/search?q=https://cdn.zabbix.com/zabbix/binaries/stable/7.4/7.4.5/zabbix_agent2-7.4.5-windows-amd64-openssl.msi).
2.  Place ce fichier dans un dossier partagé sur ton réseau accessible en lecture par tes serveurs (ex: `\\ton-ad-ou-filer\Deploy\Zabbix\`).
      * *Note :* Assure-toi que le groupe "Ordinateurs du domaine" (Domain Computers) a les droits de **Lecture** sur ce partage.

### 2\. Création du Script d'Installation (PowerShell)

Nous allons créer un petit script qui vérifie si Zabbix est là. S'il n'est pas là, il l'installe avec la bonne config (IP serveur et Mode Actif).

Crée un fichier nommé `Install-ZabbixAgent.ps1` dans le même dossier partagé :

```powershell
# --- CONFIGURATION ---
$ZabbixServerIP = "192.168.20.25"
$MsiPath = "\\ton-ad-ou-filer\Deploy\Zabbix\zabbix_agent2-7.4.5-windows-amd64-openssl.msi"
$LogFile = "C:\Windows\Temp\zabbix_install.log"

# Vérifier si le service Zabbix Agent 2 existe déjà
$Service = Get-Service -Name "Zabbix Agent 2" -ErrorAction SilentlyContinue

If (-Not $Service) {
    Try {
        Write-Output "Installation de Zabbix Agent 2..." | Out-File $LogFile
        
        # Arguments MSI pour le Mode Actif :
        # SERVERACTIVE : L'IP pour les checks actifs
        # SERVER : L'IP pour les checks passifs (ou 127.0.0.1 si tu veux bloquer) et commandes distantes
        # HOSTNAME : Si omis, l'agent prend automatiquement le nom Windows (recommandé)
        # ENABLEPATH : Ajoute les binaires au PATH Windows
        
        $Args = "/i `"$MsiPath`" /qn /norestart SERVERACTIVE=$ZabbixServerIP SERVER=$ZabbixServerIP ENABLEPATH=1"
        
        Start-Process -FilePath "msiexec.exe" -ArgumentList $Args -Wait -NoNewWindow
        
        Write-Output "Installation terminée avec succès." | Out-File $LogFile -Append
    }
    Catch {
        Write-Output "Erreur lors de l'installation : $_" | Out-File $LogFile -Append
    }
} Else {
    Write-Output "L'agent Zabbix est déjà installé. Aucune action requise." | Out-File $LogFile
}
```

*Remplace `\\ton-ad-ou-filer\...` par ton vrai chemin réseau.*

### 3\. Création de la GPO

1.  Ouvre la console **Gestion de stratégie de groupe** (`gpmc.msc`).
2.  Crée une nouvelle GPO nommée **"Deploy - Zabbix Agent 2"**.
3.  Fais un clic droit \> **Modifier**.

#### A. Configurer le Script de Démarrage

1.  Va dans : **Configuration ordinateur** \> **Stratégies** \> **Paramètres Windows** \> **Scripts (démarrage/arrêt)**.
2.  Double-clique sur **Démarrage** (Startup).
3.  Va dans l'onglet **Scripts PowerShell**.
4.  Clique sur **Ajouter...** et indique le chemin réseau complet de ton script : `\\ton-ad-ou-filer\Deploy\Zabbix\Install-ZabbixAgent.ps1`.
5.  Valide.

#### B. Configurer le Pare-feu Windows (Crucial)

Comme pour Linux, il faut ouvrir le port pour que le serveur puisse contacter l'agent (commandes distantes) et autoriser l'agent à sortir.

1.  Va dans : **Configuration ordinateur** \> **Stratégies** \> **Paramètres Windows** \> **Paramètres de sécurité** \> **Pare-feu Windows Defender avec fonctions avancées**.
2.  Dossier **Règles de trafic entrant** (Inbound Rules) \> Clic droit \> **Nouvelle règle**.
      * **Type de règle :** Port.
      * **Protocole :** TCP / Ports locaux spécifiques : `10050`.
      * **Action :** Autoriser la connexion.
      * **Profil :** Domaine (suffisant si tes serveurs sont dans le domaine).
      * **Nom :** "Zabbix Agent - Port 10050".
3.  *(Sécurité)* Une fois la règle créée, double-clique dessus, va dans l'onglet **Étendue** (Scope).
      * Dans "Adresse IP distante", coche "Ces adresses IP" et ajoute `192.168.20.25`. Ainsi, seul ton serveur Zabbix peut interroger tes serveurs Windows.

### 4\. Déploiement et Test

1.  Lie (Link) ta GPO à l'**Unité d'Organisation (OU)** où se trouvent tes serveurs Windows.
2.  Sur un serveur de test, lance une invite de commande en admin et tape :
    ```cmd
    gpupdate /force
    ```
3.  Redémarre le serveur (nécessaire car c'est un script de démarrage "Computer").
4.  Après le redémarrage, vérifie :
      * Dans `services.msc` : Le service "Zabbix Agent 2" doit être "En cours d'exécution".
      * Le fichier `C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf` : Vérifie qu'il contient bien `ServerActive=192.168.20.25`.

### Résumé des paramètres MSI utilisés

L'avantage de cette méthode, c'est que l'installateur MSI génère le fichier de config pour toi :

| Paramètre MSI | Valeur | Effet dans `zabbix_agent2.conf` |
| :--- | :--- | :--- |
| `SERVERACTIVE` | `192.168.20.25` | Active le mode Actif vers cette IP. |
| `SERVER` | `192.168.20.25` | Autorise cette IP à faire des requêtes passives. |
| `HOSTNAME` | *(Non défini)* | Zabbix utilisera le `ComputerName` Windows automatiquement. |
