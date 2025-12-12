# Darkly — README : Vulnérabilité de parcours de répertoires / Inclusion de fichiers locaux (LFI)

## Résumé
Ce document décrit une vulnérabilité de type **Directory Traversal / Local File Inclusion (LFI)** découverte dans l’application Darkly.  
Le problème permet à un utilisateur de contrôler partiellement un chemin de fichier via le paramètre `page`, ce qui peut conduire à la lecture de fichiers locaux arbitraires par l’application web.

Ce README explique l’origine du problème, son impact, sa détection et les correctifs recommandés, de manière factuelle et technique.

---

## Vue d’ensemble de la vulnérabilité
- **Type :** Directory Traversal / Local File Inclusion (LFI)
- **Paramètre affecté :** `page` (paramètre HTTP GET)
- **Comportement observé :** la fourniture d’un chemin contenant des séquences `..` permet à l’application d’inclure ou de lire des fichiers situés en dehors du répertoire prévu.
- **Exemple observé :**
  ```
  http://10.14.200.249/?page=../../../../../../../etc/passwd
  ```
  Cette requête a retourné le contenu du fichier `/etc/passwd`.
- **Risque :** divulgation de fichiers locaux lisibles par le processus web, pouvant mener à des fuites d’informations sensibles, voire à une exécution de code à distance dans le cadre d’attaques chaînées.

---

## Preuve de concept (niveau élevé)
Une requête HTTP contenant un paramètre `page` avec plusieurs segments `..` a permis de résoudre le chemin jusqu’à la racine du système et d’afficher le contenu de fichiers système sensibles.

La présence du contenu de `/etc/passwd` confirme que l’entrée utilisateur est utilisée directement (ou insuffisamment validée) lors de la construction du chemin de fichier.

> Remarque : ce document ne constitue pas un guide d’exploitation. Il vise uniquement à faciliter la compréhension et la correction de la faille.

---

## Analyse de la cause racine
1. **Chemin de fichier contrôlé par l’utilisateur sans validation**
   - L’application construit un chemin système à partir du paramètre `page` sans validation ni normalisation adéquate.

2. **Absence de liste blanche (allowlist)**
   - Aucun mécanisme strict ne limite les pages ou templates autorisés.
   - Une logique de filtrage par liste noire (ex. suppression de `..`) est insuffisante et fragile.

3. **Séparation de privilèges insuffisante**
   - Le processus web dispose de droits de lecture sur des fichiers situés en dehors du répertoire applicatif.

4. **Manque de défense en profondeur**
   - Journalisation, surveillance et restrictions d’accès aux fichiers insuffisantes pour détecter ou limiter les abus.

---

## Impact
- **Divulgation d’informations**
  - Lecture de fichiers de configuration, secrets applicatifs, clés SSH, code source, etc.
- **Élévation indirecte des privilèges / exécution de code**
  - En présence d’autres failles (écriture de fichiers, interprétation de code inclus), cette vulnérabilité peut être chaînée vers une exécution de code à distance.
- **Risques réglementaires et de conformité**
  - Exposition potentielle de données sensibles ou personnelles.

---

## Sévérité (estimation technique)
En considérant la facilité d’exploitation, l’impact potentiel et les possibilités de chaînage, cette vulnérabilité doit être considérée comme de **sévérité élevée** tant qu’elle n’est pas corrigée et validée.

---

## Remédiations recommandées
Les mesures suivantes sont classées des plus robustes aux plus fragiles. Une approche en couches est fortement recommandée.

### 1. Implémenter une liste blanche explicite
- Mapper des identifiants logiques (`home`, `about`, `contact`) vers des templates statiques.
- Ne jamais accepter de chemins de fichiers bruts provenant de l’utilisateur.

### 2. Canonicaliser et valider les chemins
- Utiliser une résolution de chemin canonique (`realpath` ou équivalent).
- Vérifier que le chemin final se situe strictement dans le répertoire autorisé.
- Rejeter toute requête qui en sort.

### 3. Éviter l’inclusion directe de chaînes contrôlées par l’utilisateur
- Ne pas passer d’entrée utilisateur brute à des fonctions `include`, `require` ou de lecture de fichiers.
- Utiliser des moteurs de templates sécurisés.

### 4. Réduire les privilèges du processus web
- Exécuter le serveur sous un compte dédié avec des droits minimaux.
- Restreindre l’accès en lecture aux seuls fichiers nécessaires.

### 5. Durcir la configuration serveur
- Empêcher l’accès aux fichiers hors de la racine web.
- Désactiver le listing de répertoires et les handlers inutiles.

### 6. Préférer la canonicalisation aux listes noires
- Les filtres basés sur des motifs (`..`, `../`) sont facilement contournables.
- Toujours valider le chemin final résolu.

### 7. Journalisation et surveillance
- Journaliser les tentatives d’accès anormales.
- Déclencher des alertes sur des schémas suspects (accès répétés hors templates).

### 8. Utiliser un moteur de templates sécurisé
- Séparer strictement l’identifiant logique du template et le chemin système.
- Éviter toute évaluation de contenu comme du code exécutable.

---

## Conclusion
Cette vulnérabilité illustre un cas classique de **Directory Traversal / LFI** dû à une confiance excessive dans une entrée utilisateur.  
Une conception basée sur des listes blanches, une validation stricte des chemins et le principe du moindre privilège permet d’éliminer efficacement ce type de faille.
