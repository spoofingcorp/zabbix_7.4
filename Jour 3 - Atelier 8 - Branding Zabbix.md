# Jour 3 – Atelier 8

# **Branding Zabbix (Logo personnalisé & Identité visuelle)**

Objectif : Personnaliser l’interface Zabbix 7.4 avec votre logo (entreprise, client ou service NOC) et appliquer cette identité visuelle sur :

* l’écran de connexion,
* la barre supérieure du frontend,
* les rapports PDF (Reporting planifié),
* les tableaux de bord.

Le branding renforce l’aspect professionnel et facilite l’adoption de la plateforme par les équipes.

---

# 1. Prérequis

Avant de commencer, préparer :

### 1.1 Fichiers PNG

* Format recommandé : **PNG transparent**
* Résolution idéale :

  * **Logo principal :** 300×80 px
  * **Logo PDF :** 600×200 px
* Taille < **200 Ko** (pour éviter les temps de chargement excessifs).

### 1.2 Droit administrateur

L’utilisateur doit disposer du rôle : **Super Admin**.

---

# 2. Accéder à la configuration de Branding

**Chemin :**
*Administration → General → Branding*

Vous y trouverez trois sections principales :

1. **Login screen branding**
2. **Navigation bar branding**
3. **PDF report branding**

Chaque section permet d’ajouter un logo différent selon le cas d’usage.

---

# 3. Personnalisation de l’écran de connexion (Login Screen)

C’est l’élément vu par tous les utilisateurs avant authentification.

Dans la section **Login screen branding** :

1. **Logo file :**
   Importer votre fichier PNG (300×80 recommandé).
2. **Product name :**
   Exemple : `Digital Chemists Monitoring Portal`

   > Remplace “Zabbix” sous le logo.
3. **Slogan (optionnel) :**
   Exemple : `Infrastructure & Network Monitoring – NOC 24/7`
4. **Copyright text :**
   Exemple : `© 2025 Digital Chemists – All rights reserved.`
5. Cliquer sur **Apply**.

Résultat :

* Logo et nom produits visibles sur la page de connexion.
* Identité visuelle professionnelle dès l’accueil.

---

# 4. Barre supérieure du Frontend (Navigation Bar)

Cette zone correspond au bandeau du haut visible après connexion.
Le branding ici est très utile pour les environnements multi-clients ou multi-équipes.

Dans la section **Navigation bar branding** :

1. **Logo file :**
   Importer une version compacte (ex : 150×40 px).
2. **Product name :**
   Exemple : `Zabbix DC-Monitoring`
3. Cliquer sur **Apply**.

Effet :
Le logo apparaît dans le coin supérieur gauche à la place du logo Zabbix par défaut.

---

# 5. Branding des exports et Reporting PDF

Cette section concerne les rapports PDF générés automatiquement (Scheduled reports), ainsi que les exports manuels.

Dans la section **PDF report branding** :

1. **Logo file :**
   Ajouter un logo haute résolution (600×200 px recommandé).
2. **Report title :**
   Exemple : `Rapport hebdomadaire de supervision – Digital Chemists`
3. **Subtitle (optionnel) :**
   Exemple : `Cockpit – État global des systèmes`
4. Cliquer sur **Apply**.

Résultat dans les PDF :

* Logo propre en haut à gauche,
* Titre professionnel,
* Mise en page corporate.

---

# 6. Vérification du branding

## 6.1 Login

Déconnecter et vérifier l’écran d’authentification.

## 6.2 Interface

Naviguer dans l’interface → le logo doit apparaître dans le bandeau supérieur.

## 6.3 Rapport PDF

Effectuer un export PDF depuis n’importe quel dashboard :

*Monitoring → Dashboards → … → Export to PDF*

Le logo doit apparaître en en-tête du document.

---

# 7. Bonnes Pratiques Branding

### 7.1 Préparer plusieurs versions de logos

* Variante horizontale
* Variante compacte
* Variante haute résolution (PDF)

### 7.2 Vérifier la lisibilité sur fond sombre

Depuis Zabbix 6+, le thème **Dark Mode** est courant.

### 7.3 Unifier les intitulés

Utiliser le même nom produit sur tous les écrans.

### 7.4 Adapter le branding pour chaque client

Pour un MSP, créer une identité par tenant :

* Branding A pour “Client A – Supervision”
* Branding B pour “Client B – Monitoring Portal”

---

# 8. Résultat final

Après application du branding :

* La plateforme présente l’identité visuelle de l’entreprise ou du client.
* Les PDF envoyés automatiquement sont professionnels et prêts à être transmis au COMEX ou aux équipes techniques.
* Les dashboards diffusés sur TV NOC sont alignés avec la charte graphique.

