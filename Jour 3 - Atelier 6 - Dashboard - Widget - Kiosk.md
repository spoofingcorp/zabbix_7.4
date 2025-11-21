# Jour 3 – Atelier 6

# **Création d’un Dashboard Opérationnel (Cockpit) – Zabbix 7.4**

Objectif : Mettre en place un tableau de bord en temps réel destiné à un écran d’exploitation (TV ou grand moniteur), en utilisant les widgets avancés de Zabbix 7.4 dont **Honeycomb**.

---

# 1. Création du Dashboard

**Chemin :** *Monitoring → Dashboards* → **Create dashboard**

Renseigner les champs suivants :

1. **Owner :** Administrateur (ou votre compte).
2. **Name :** `🔴 Cockpit – Supervision Live`.
3. **Default page display period :** `30 seconds`

   > Ce paramètre définit la durée d’affichage d’une **page** en mode slideshow
   > (il ne s’agit pas du rafraîchissement des widgets).
4. Cliquer sur **Apply**.

Une page vide s’ouvre en mode édition.

---

# 2. Widget n°1 – Liste des Problèmes Actifs

Ce widget représente la « to-do list » opérationnelle en temps réel.

1. Cliquer sur **Add widget**.
2. **Type :** `Problems`.
3. **Name :** `🔥 Alertes en cours`.
4. **Show :** `Recent problems`

   > Affiche les problèmes actifs ainsi que ceux résolus récemment pour suivi.
5. **Show tags :** `1`

   > Affiche la première colonne de tags (utile pour la répartition par équipe).
6. **Show operational data :** `Separately`

   > Présente la valeur actuelle (ex. « 93 % used ») dans une colonne dédiée.
7. **Advanced filter (facultatif) :**

   * Tag name : `Team`
   * Tag value : `SysAdmin`
8. Cliquer sur **Add**.

---

# 3. Widget n°2 – Honeycomb (État Global des Disques)

Le widget Honeycomb permet une visualisation synthétique de l’état de dizaines de disques simultanément.

1. Cliquer sur **Add widget**.
2. **Type :** `Honeycomb`.
3. **Name :** `💾 État des disques (Global)`.
4. **Host groups :** `Linux servers`
5. **Item patterns :** `Used space percentage`

   > Ce pattern correspond aux items de type :
   > `vfs.fs.dependent.size[{#FSNAME},pused]`
6. **Show values :** coché
7. **Thresholds :**

   * `80` → couleur Orange (Alerte)
   * `90` → couleur Rouge (Critique)
8. **Refresh interval :** `30 seconds`

   > Ce paramètre définit le rafraîchissement réel des données.
9. Cliquer sur **Add**.

---

# 4. Widget n°3 – Top Hosts (Charges CPU)

Ce widget permet d’identifier les hôtes les plus chargés.

1. Cliquer sur **Add widget**.
2. **Type :** `Top hosts`.
3. **Name :** `🏆 Top CPU Load`.
4. **Host groups :** `Linux servers`.
5. **Columns → Add :**

   * **Name :** `CPU Load`
   * **Data :** `Item value`
   * **Item :** `CPU load (1 min average)`
     (clé : `system.cpu.load[all,avg1]`)
   * **Display :** `Bar`
   * **Thresholds :**

     * `3` → Orange
     * `5` → Rouge
6. **Order by :** Column `CPU Load`.
7. **Order :** `Top N`.
8. **Host count :** `5`.
9. **Refresh interval :** `30 seconds`.
10. Cliquer sur **Add**.

---

# 5. Rafraîchissement et Mode Kiosk

## 5.1 Rafraîchissement manuel

Il est possible de modifier temporairement l’intervalle sans reconfigurer les widgets :

* Cliquer sur l’icône **Horloge** en haut à droite.
* Choisir un intervalle (ex. `10 seconds`, `1 minute`).

## 5.2 Mode plein écran (Kiosk)

Pour l’affichage sur un écran au sein d’une salle d’exploitation :

1. Cliquer sur **…** (menu en haut à droite).
2. Sélectionner **Kiosk mode**.

   > En mode kiosk, l’interface disparaît et ne reste que le contenu.
   > Pour quitter : retirer `&kiosk=1` dans l’URL ou bouger la souris dans le coin supérieur.

---

# 6. Résumé des widgets

| Widget        | Rôle                                         | Particularité (Zabbix 7.4) |
| ------------- | -------------------------------------------- | -------------------------- |
| **Problems**  | Liste d’alertes en temps réel                | Standard                   |
| **Honeycomb** | Vue synthétique par seuils (disques, ports…) | **Nouveau widget visuel**  |
| **Top hosts** | Analyse capacitaire (CPU, RAM…)              | Amélioré en 7.x            |


