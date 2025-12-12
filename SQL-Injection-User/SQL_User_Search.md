# Darkly — README : Injection SQL menant à la divulgation d’identifiants

## Résumé
Ce document décrit une vulnérabilité d’**injection SQL** affectant la fonctionnalité de recherche d’utilisateurs. En injectant des fragments SQL spécialement conçus dans le paramètre de recherche, il est possible d’énumérer la structure de la base de données, d’extraire des champs sensibles, de récupérer des identifiants hachés et, in fine, d’obtenir la flag.

Ce write-up explique le fonctionnement de la faille, ses causes et les correctifs appropriés.

---

## Vue d’ensemble de la vulnérabilité
- **Type :** Injection SQL (basée sur `UNION`)
- **Fonctionnalité affectée :** Formulaire de recherche d’utilisateurs
- **Vecteur d’attaque :** Entrée utilisateur non filtrée concaténée directement dans une requête SQL
- **Impact :** Accès en lecture aux tables de la base de données ; divulgation d’identifiants

---

## Preuve de concept (niveau élevé)

1. Le champ de recherche accepte une injection SQL directe. En entrant :
   ```
   1 UNION SELECT table_name, column_name FROM information_schema.columns
   ```
   l’application fusionne les résultats contrôlés par l’attaquant avec ceux de la requête légitime.

2. Cela expose le schéma de la base, notamment la table `users` et des colonnes comme `countersign` et `Commentaire`.

3. À partir de ces informations, le payload suivant permet d’extraire des données sensibles :
   ```
   1 UNION SELECT countersign, Commentaire FROM users
   ```

4. L’application affiche alors :
   ```
   First name: 5ff9d0165b4f92b14994e5c685cdce28
   Surname: Decrypt this password -> then lower all the char. Sh256 on it and it's good !
   ```

5. Le hash `5ff9d0165b4f92b14994e5c685cdce28` correspond au texte en clair `FortyTwo`.

6. Après conversion en minuscules puis hachage SHA‑256, on obtient la flag :
   ```
   10a16d834f9b1e4068b25c4c46fe0284e99e44dceaf08098fc83925ba6310ff5
   ```

Cela confirme que l’entrée utilisateur est directement concaténée dans les requêtes SQL, sans paramétrage.

---

## Analyse de la cause racine

1. **Construction dynamique de requêtes SQL avec des données utilisateur**
   - Exemple typique :
     ```sql
     SELECT first_name, last_name FROM users WHERE id = '<input_utilisateur>';
     ```
   - Sans échappement ni liaison de paramètres, les fragments injectés sont exécutés.

2. **Absence de validation côté serveur**
   - Les entrées ne sont pas contraintes (ex. valeurs numériques attendues).

3. **Utilisation non restreinte de `UNION`**
   - Les résultats de la base sont renvoyés directement à l’interface.

4. **Exposition du schéma interne**
   - Accès non restreint à `information_schema`.

5. **Privilèges SQL trop larges**
   - L’utilisateur SQL de l’application dispose de droits de lecture étendus.

6. **Absence de filtrage en sortie**
   - L’affichage direct des champs facilite l’exfiltration de données.

---

## Impact
- **Énumération du schéma de la base de données**
- **Divulgation d’identifiants hachés**
- **Cassage hors ligne de mots de passe** si des algorithmes faibles sont utilisés
- **Élévation de privilèges et chaînage d’attaques**
- **Compromission complète de la confidentialité des données**

Sévérité estimée : **Critique**, en raison d’un accès en lecture étendu à la base de données.

---

## Remédiations recommandées

### 1. Utiliser des requêtes préparées (paramétrées)
Remplacer toute construction dynamique par des requêtes avec paramètres :
```
SELECT * FROM users WHERE id = ?
```

### 2. Appliquer une validation stricte des entrées
- Restreindre les champs aux types attendus (ex. identifiants numériques).
- Rejeter toute entrée malformée avant l’accès à la base.

### 3. Appliquer le principe du moindre privilège
- Interdire l’accès à `information_schema`.
- Limiter les droits SQL aux seules tables et colonnes nécessaires.

### 4. Masquer les erreurs internes
- Ne jamais afficher d’erreurs SQL ou de sorties brutes côté client.

### 5. Supprimer les indices sensibles des champs
- Les messages du type “Decrypt this password...” ne doivent pas exister en base.

### 6. Renforcer le stockage des mots de passe
- Remplacer les algorithmes faibles par :
  - Argon2
  - bcrypt
  - PBKDF2
- Utiliser des sels uniques.

### 7. Ajouter surveillance et limitation
- Détecter les tentatives répétées d’énumération.
- Journaliser et alerter sur les patterns suspects.

---

## Détection et vérification
- **Avant correction :** Les payloads `'` et `UNION SELECT` fonctionnent.
- **Après correction :**
  - Les entrées invalides sont rejetées.
  - Les injections `UNION` échouent.
  - Les erreurs SQL ne sont plus visibles.
  - L’accès à `information_schema` est bloqué.

---

## Checklist de validation
- [ ] Toutes les requêtes sont paramétrées.
- [ ] Validation serveur des entrées appliquée.
- [ ] Privilèges SQL restreints.
- [ ] Aucune sortie brute de la base.
- [ ] Hachage des mots de passe renforcé.
- [ ] Tests confirmant l’absence de SQLi.

---

## Recommandations complémentaires
1. Auditer toutes les requêtes SQL dynamiques.
2. Mettre en place des règles WAF contre les patterns SQLi.
3. Automatiser des tests d’injection SQL réguliers.
4. Appliquer des revues de code orientées sécurité.

---
