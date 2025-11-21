# Ansible Playbook - Install Agent2 mode active

Playbook Ansible permettant de détecter automatiquement la famille de l'OS (Debian/Ubuntu ou RHEL/CentOS/Rocky) et installe le bon dépôt de l'agent2 Zabbix en mode actif.

Voici le playbook complet et optimisé pour le **mode Actif**.

### Prérequis

- Assure-toi que ton fichier d'inventaire (`hosts`) contient tes serveurs cibles.
- Règles de firewall 10050 et 10051 (Si besoin voir version - Ansible Playbook Agent2 Zabbix + Firewall Rules)

### Le Playbook (`install_zabbix_agent2.yml`)

Copie ce code dans un fichier YAML.

```yaml
---
- name: Installation et Configuration de Zabbix Agent 2 (Mode Actif)
  hosts: all
  become: yes
  vars:
    # --- CONFIGURATION ---
    zabbix_server_ip: "192.168.20.25"  # Adresse IP du serveur Zabbix (Mode Actif)
    # Attention: Vérifie que le dépôt 7.4 existe pour ton OS. 
    # Sinon, remplace par "7.0" ou "6.4"
    zabbix_version: "7.0" 
    
    # Le nom d'hôte doit correspondre EXACTEMENT à celui déclaré dans l'interface web Zabbix
    system_hostname: "{{ inventory_hostname }}"

  tasks:
    # ---------------------------------------------------------
    # 1. DÉTECTION ET INSTALLATION DES DÉPÔTS (REPO)
    # ---------------------------------------------------------
    
    # Partie pour les systèmes basés sur Debian / Ubuntu
    - name: "Debian/Ubuntu | Téléchargement du paquet release Zabbix"
      ansible.builtin.get_url:
        url: "https://repo.zabbix.com/zabbix/{{ zabbix_version }}/{{ ansible_distribution | lower }}/pool/main/z/zabbix-release/zabbix-release_{{ zabbix_version }}-2+{{ ansible_distribution_release }}_all.deb"
        dest: "/tmp/zabbix-release.deb"
      when: ansible_os_family == "Debian"

    - name: "Debian/Ubuntu | Installation du dépôt Zabbix"
      ansible.builtin.apt:
        deb: "/tmp/zabbix-release.deb"
        state: present
      when: ansible_os_family == "Debian"

    - name: "Debian/Ubuntu | Mise à jour du cache apt"
      ansible.builtin.apt:
        update_cache: yes
      when: ansible_os_family == "Debian"

    # Partie pour les systèmes basés sur RHEL / Rocky / Alma / CentOS
    - name: "RHEL/Rocky | Installation du dépôt Zabbix"
      ansible.builtin.dnf:
        name: "https://repo.zabbix.com/zabbix/{{ zabbix_version }}/rhel/{{ ansible_distribution_major_version }}/x86_64/zabbix-release-{{ zabbix_version }}-2.el{{ ansible_distribution_major_version }}.noarch.rpm"
        state: present
        disable_gpg_check: yes
      when: ansible_os_family == "RedHat"

    # ---------------------------------------------------------
    # 2. INSTALLATION DU PAQUET
    # ---------------------------------------------------------
    - name: "Installation de l'agent Zabbix 2"
      ansible.builtin.package:
        name: zabbix-agent2
        state: present

    # ---------------------------------------------------------
    # 3. CONFIGURATION (MODE ACTIF)
    # ---------------------------------------------------------
    - name: "Configuration | Définir ServerActive (Mode Actif)"
      ansible.builtin.lineinfile:
        path: /etc/zabbix/zabbix_agent2.conf
        regexp: '^ServerActive='
        line: "ServerActive={{ zabbix_server_ip }}"
        state: present

    - name: "Configuration | Définir le Hostname (Crucial pour le mode actif)"
      ansible.builtin.lineinfile:
        path: /etc/zabbix/zabbix_agent2.conf
        regexp: '^Hostname='
        line: "Hostname={{ system_hostname }}"
        state: present

    # Optionnel : Commenter le mode passif pour forcer l'actif pur (sécurité)
    # Tu peux retirer cette tâche si tu veux garder le mode passif disponible
    - name: "Configuration | Désactiver le mode Passif (Server=)"
      ansible.builtin.lineinfile:
        path: /etc/zabbix/zabbix_agent2.conf
        regexp: '^Server='
        line: "# Server=127.0.0.1"
        state: present

    # ---------------------------------------------------------
    # 4. DÉMARRAGE DU SERVICE
    # ---------------------------------------------------------
    - name: "Service | Activer et démarrer Zabbix Agent 2"
      ansible.builtin.service:
        name: zabbix-agent2
        state: started
        enabled: yes
```

### Explications Techniques

1.  **Détection de l'OS (`ansible_os_family`)** :

      * Le playbook utilise les "facts" d'Ansible. Il sait automatiquement si la machine cible est une `Debian` (incluant Ubuntu) ou `RedHat` (incluant Rocky, Alma, CentOS).
      * Il construit l'URL de téléchargement dynamiquement en utilisant `{{ ansible_distribution_release }}` (ex: *jammy*, *bookworm*) ou `{{ ansible_distribution_major_version }}` (ex: *8*, *9*).

2.  **Configuration Mode Actif (`zabbix_agent2.conf`)** :

      * **`ServerActive`** : C'est la directive clé. L'agent va initier la connexion vers ton serveur `192.168.20.25` (port 10051 par défaut) pour demander sa configuration.
      * **`Hostname`** : En mode actif, le serveur Zabbix ne se fie pas à l'IP de l'agent, mais à son nom. **Le nom défini ici doit être identique** au champ "Host name" dans l'interface web de Zabbix. J'ai utilisé `inventory_hostname` (le nom dans ton fichier hosts Ansible).

3.  **Version Zabbix** :

      * J'ai mis une variable `zabbix_version: "7.0"`. **Note importante :** Si tu utilises vraiment la 7.4, change cette valeur. Cependant, vérifie que l'URL du dépôt existe. Souvent, on installe l'agent 7.0 LTS même si le serveur est en 7.4, car la compatibilité ascendante est bonne.

### Comment lancer le playbook

1.  Crée ton fichier d'inventaire `hosts.ini` :

    ```ini
    [linux_servers]
    192.168.20.50
    192.168.20.51
    mon-serveur-web.local
    ```

2.  Lance la commande :

    ```bash
    ansible-playbook -i hosts.ini install_zabbix_agent2.yml
    ```



___________________________________________________________________________

# Playbook version - Ansible Playbook Agent2 Zabbix + Firewall Rules

Ajout d'une section **5. SÉCURITÉ / PARE-FEU** qui détecte si tu utilises `ufw` (Debian/Ubuntu) ou `firewalld` (RHEL/CentOS) et applique une règle stricte : **Seul le serveur Zabbix (192.168.20.25) a le droit d'entrer.**

### Le Playbook Mis à Jour (`install_zabbix_agent2_securise.yml`)

```yaml
---
- name: Installation Zabbix Agent 2 (Actif) avec Pare-feu
  hosts: all
  become: yes
  vars:
    zabbix_server_ip: "192.168.20.25"
    zabbix_version: "7.0" # Mets "7.0" pour la compatibilité ou "6.4" si 7.4 n'a pas de repo
    system_hostname: "{{ inventory_hostname }}"

  tasks:
    # ... [LES ÉTAPES 1, 2, 3 et 4 SONT IDENTIQUES AU PRÉCÉDENT PLAYBOOK] ...
    # Je les réintègre brièvement pour que tu aies le fichier complet
    
    # 1. REPOS
    - name: "Debian/Ubuntu | Setup Repo"
      ansible.builtin.apt:
        deb: "https://repo.zabbix.com/zabbix/{{ zabbix_version }}/{{ ansible_distribution | lower }}/pool/main/z/zabbix-release/zabbix-release_{{ zabbix_version }}-2+{{ ansible_distribution_release }}_all.deb"
      when: ansible_os_family == "Debian"

    - name: "Debian/Ubuntu | Update Cache"
      ansible.builtin.apt: 
        update_cache: yes
      when: ansible_os_family == "Debian"

    - name: "RHEL/Rocky | Setup Repo"
      ansible.builtin.dnf:
        name: "https://repo.zabbix.com/zabbix/{{ zabbix_version }}/rhel/{{ ansible_distribution_major_version }}/x86_64/zabbix-release-{{ zabbix_version }}-2.el{{ ansible_distribution_major_version }}.noarch.rpm"
        disable_gpg_check: yes
      when: ansible_os_family == "RedHat"

    # 2. INSTALLATION
    - name: "Installation Agent"
      ansible.builtin.package:
        name: zabbix-agent2
        state: present

    # 3. CONFIGURATION
    - name: "Config | ServerActive"
      ansible.builtin.lineinfile:
        path: /etc/zabbix/zabbix_agent2.conf
        regexp: '^ServerActive='
        line: "ServerActive={{ zabbix_server_ip }}"

    - name: "Config | Hostname"
      ansible.builtin.lineinfile:
        path: /etc/zabbix/zabbix_agent2.conf
        regexp: '^Hostname='
        line: "Hostname={{ system_hostname }}"

    # 4. SERVICE
    - name: "Service | Start Agent"
      ansible.builtin.service:
        name: zabbix-agent2
        state: started
        enabled: yes

    # ---------------------------------------------------------
    # 5. SÉCURITÉ / PARE-FEU (NOUVELLE SECTION)
    # ---------------------------------------------------------
    
    # --- CAS 1 : UFW (Ubuntu / Debian) ---
    - name: "Firewall (UFW) | Autoriser le Serveur Zabbix sur le port 10050"
      community.general.ufw:
        rule: allow
        proto: tcp
        port: '10050'
        src: "{{ zabbix_server_ip }}"
        comment: "Allow Zabbix Server connection"
      when: 
        - ansible_os_family == "Debian"
        # On vérifie si UFW est installé pour éviter une erreur
        - ansible_facts.packages['ufw'] is defined 

    # --- CAS 2 : Firewalld (RHEL / Rocky / Alma) ---
    - name: "Firewall (Firewalld) | Autoriser le Serveur Zabbix (Rich Rule)"
      ansible.posix.firewalld:
        rich_rule: 'rule family="ipv4" source address="{{ zabbix_server_ip }}" port port="10050" protocol="tcp" accept'
        permanent: yes
        state: enabled
        immediate: yes # Applique la règle tout de suite sans reload manuel
        zone: public
      when: 
        - ansible_os_family == "RedHat"
```

### Prérequis pour que ça marche

Ansible a besoin de "collections" spécifiques pour gérer les pare-feux. Il est fort probable que tu les aies déjà, mais si tu as une erreur du type `module not found`, lance cette commande sur ta machine Ansible (celle où tu lances le playbook) :

```bash
ansible-galaxy collection install community.general ansible.posix
```

### Points clés de cette configuration

1.  **Sécurité maximale (Whitelisting)** :

      * Sur **UFW**, on utilise le paramètre `src: "{{ zabbix_server_ip }}"`.
      * Sur **Firewalld**, on utilise une `rich_rule`.
      * Cela signifie que si quelqu'un d'autre sur le réseau essaie de scanner le port 10050 de tes agents, il sera rejeté. Seul ton serveur `192.168.20.25` passera.

2.  **Détection des paquets** :

      * Pour UFW, j'ai ajouté une petite condition de sécurité `ansible_facts.packages['ufw'] is defined`. Cela évite que le playbook ne plante sur une Debian "minimale" qui n'aurait pas UFW installé. (Note : il faut lancer le playbook avec l'option "gather facts" activée, ce qui est le cas par défaut).

