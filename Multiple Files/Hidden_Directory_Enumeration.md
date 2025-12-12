# Darkly — README : Énumération de répertoires cachés via `robots.txt` et récupération récursive

## Résumé
Ce document décrit une vulnérabilité dans laquelle des fichiers sensibles sont exposés via un répertoire caché profondément imbriqué, référencé dans le fichier `robots.txt`.  
Bien que `robots.txt` soit destiné à masquer du contenu aux moteurs de recherche, il révèle publiquement l’existence du répertoire `.hidden`. En énumérant récursivement ses sous‑répertoires, il est possible de récupérer de nombreux fichiers `README`, dont l’un contient la flag.

Ce write‑up explique comment la faille est découverte, pourquoi elle existe et quelles sont les stratégies de remédiation appropriées.

---

## Vue d’ensemble de la vulnérabilité
- **Type :** Divulgation d’information / Énumération de ressources prévisibles
- **Composant affecté :** répertoire `.hidden` référencé dans `robots.txt`
- **Vecteur d’attaque :** exploration récursive et récupération de fichiers
- **Impact :** exposition de documentation interne et divulgation d’une flag contenue dans un fichier `README`

---

## Preuve de concept (niveau élevé)

1. Le fichier `robots.txt` de l’application contient :
   ```
   Disallow: /.hidden
   ```
   Bien que destinée aux moteurs de recherche, cette directive **signale explicitement** l’existence du répertoire aux attaquants.

2. L’accès direct à :
   ```
   http://10.14.200.249/.hidden/
   ```
   révèle de nombreux sous‑répertoires imbriqués, chacun contenant d’autres dossiers et fichiers.

3. Pour récupérer efficacement l’ensemble des fichiers, un téléchargement récursif est effectué :
   ```
   wget -r -np -nH --cut-dirs=0 -e robots=off -U "Mozilla/5.0" http://10.14.200.249/.hidden/
   ```
   Options utilisées :
   - `-r` : téléchargement récursif
   - `-np` : ne pas remonter vers les répertoires parents
   - `-nH` : ne pas créer de dossier basé sur le nom d’hôte
   - `--cut-dirs=0` : conserver la structure des dossiers
   - `-e robots=off` : ignorer les règles `robots.txt`
   - `-U` : définir un user‑agent réaliste

4. Une fois les fichiers récupérés, tous les `README` imbriqués peuvent être inspectés :
   ```
   cat */*/*/README | grep flag
   ```

5. L’un des nombreux fichiers `README` contient la flag, récupérable sans autre logique applicative.

Cette situation démontre que l’obscurité des répertoires n’est pas un mécanisme de sécurité.

---

## Analyse de la cause racine

1. **Mauvaise utilisation de `robots.txt`**
   - `robots.txt` est un fichier **public**.
   - Lister des répertoires sensibles revient à en annoncer l’existence.

2. **Présence de fichiers sensibles dans des chemins accessibles**
   - Les fichiers internes ou liés à des challenges ne doivent jamais se trouver dans la racine web.

3. **Absence de contrôle d’accès**
   - Aucun mécanisme ne limite l’accès au répertoire `.hidden` ni à son contenu.

4. **Structure de répertoires prévisible et énumérable**
   - Les crawlers récursifs permettent une exploration exhaustive rapide.

5. **Absence de limitation ou de détection**
   - Les téléchargements massifs passent inaperçus.

---

## Impact
- **Divulgation de documentation interne** stockée dans des répertoires cachés.
- **Exposition directe de la flag** par simple énumération.
- **Scalabilité de l’attaque** : un seul crawler peut parcourir des milliers de dossiers rapidement.

Sévérité estimée : **Moyenne**, pouvant devenir **Élevée** si des identifiants réels ou des secrets sont stockés de la même manière.

---

## Remédiations recommandées

### 1. Ne jamais utiliser `robots.txt` pour masquer des ressources sensibles
`robots.txt` n’est pas un mécanisme de sécurité.

### 2. Mettre en place des contrôles d’accès côté serveur
- Interdire l’accès au répertoire `.hidden`.
- Déplacer les fichiers sensibles hors de la racine web.

### 3. Supprimer les fichiers inutiles en production
Les fichiers de debug ou de documentation interne ne doivent pas être accessibles publiquement.

### 4. Ajouter une authentification pour les ressources sensibles
Si certains fichiers doivent rester en ligne, ils doivent être protégés par une authentification.

### 5. Surveiller les tentatives d’exploration récursive
- Détecter les requêtes HTTP récursives.
- Journaliser et alerter sur les comportements anormaux.

### 6. Éviter les conventions de nommage prévisibles
Les attaquants localisent fréquemment les ressources par devinette ou énumération.

---

## Détection et vérification
- **Avant correction :**
  - `/.hidden/` est accessible.
  - Le téléchargement récursif fonctionne.
  - Un `README` contient la flag.
- **Après correction :**
  - `/.hidden/` retourne `403 Forbidden` ou `404 Not Found`.
  - Les fichiers sensibles ne sont plus accessibles.
  - Les téléchargements récursifs échouent.

---

## Checklist de validation
- [ ] Répertoire `.hidden` supprimé ou protégé.
- [ ] Fichiers sensibles déplacés hors de la racine web.
- [ ] Aucun chemin sensible référencé dans `robots.txt`.
- [ ] Règles de contrôle d’accès appliquées côté serveur.
- [ ] Les tentatives d’exploration récursive échouent correctement.

---

## Recommandations complémentaires
1. Auditer régulièrement les répertoires oubliés ou cachés.
2. Utiliser de l’automatisation pour éviter l’exposition accidentelle de fichiers internes.
3. Déployer des outils d’analyse de contenu pour détecter les documents exposés.
4. Examiner régulièrement les logs liés aux crawlers et aux accès inhabituels.

---
