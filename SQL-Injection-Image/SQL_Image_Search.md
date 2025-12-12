# Darkly — README : Injection SQL dans la fonction de recherche d’images

## Résumé
Ce document décrit une seconde vulnérabilité d’**injection SQL** affectant la fonctionnalité de recherche d’images de l’application Darkly.  
Comme pour la précédente faille SQLi, une entrée utilisateur non filtrée est intégrée directement dans une requête SQL. Cela permet à un attaquant d’énumérer la structure de la base de données, d’extraire des champs arbitraires, de récupérer des valeurs sensibles et, in fine, d’obtenir une flag.

L’attaque est similaire dans sa technique, mais cible une table différente (`list_images`) et expose une valeur assimilable à un secret.

---

## Vue d’ensemble de la vulnérabilité
- **Type :** Injection SQL (basée sur `UNION`)
- **Fonctionnalité affectée :** Barre de recherche d’images
- **Vecteur d’attaque :** Concaténation directe de l’entrée utilisateur dans une requête SQL
- **Impact :** Divulgation de métadonnées d’images, d’indices intégrés et de hachages menant à la récupération d’une flag

---

## Preuve de concept (niveau élevé)

1. Injection du payload suivant dans le champ de recherche d’images :
   ```
   1 UNION SELECT table_name, column_name FROM information_schema.columns
   ```
   Cela révèle la structure de la base de données, notamment la table `list_images` et ses colonnes, comme `comment`.

2. Exploitation de ces informations pour extraire les données de la table ciblée :
   ```
   1 UNION SELECT comment, comment FROM list_images
   ```

3. L’application affiche alors :
   ```
   Title: If you read this just use this md5 decode lowercase then sha256 to win this flag ! : 1928e8083cf461a51303633093573c46
   ```

4. Le hash MD5 `1928e8083cf461a51303633093573c46` correspond au texte en clair `albatroz`.

5. Après conversion en minuscules (déjà le cas ici) puis application d’un hash SHA‑256, on obtient la flag :
   ```
   f2a29020ef3132e01dd61df97fd33ec8d7fcd1388cc9601e7db691d17d4d6188
   ```

Cela confirme un accès en lecture complète aux contenus des tables via l’injection SQL.

---

## Analyse de la cause racine

1. **Construction dynamique de requêtes SQL**
   - L’entrée contrôlée par l’utilisateur est directement injectée dans la requête SQL.

2. **Absence de requêtes préparées**
   - Le backend ne sépare pas le code SQL des données via des paramètres.

3. **Réflexion directe des valeurs de la base dans l’interface**
   - Combinée à l’injection SQL, cette pratique facilite l’exfiltration.

4. **Exposition étendue du schéma**
   - L’accès à `information_schema` permet de cartographier l’intégralité de la base.

5. **Stockage inapproprié de données sensibles**
   - La présence d’indices et de hachages MD5 dans le champ `comment` montre un mélange entre métadonnées et données sensibles.

---

## Impact
- **Extraction arbitraire de données** accessibles par l’utilisateur SQL.
- **Divulgation de hachages**, permettant un cassage hors ligne.
- **Chaînage avec d’autres vulnérabilités**, menant à des escalades de privilèges.
- **Perte de confidentialité** sur les données et la logique applicative.

Sévérité estimée : **Critique**, car un accès en lecture étendu à la base de données est possible.

---

## Remédiations recommandées

### 1. Utiliser des requêtes SQL paramétrées
Toutes les interactions avec la base doivent utiliser des requêtes préparées, par exemple :
```
SELECT title, comment FROM list_images WHERE id = ?
```

### 2. Valider et filtrer les entrées
- Restreindre le champ de recherche aux formats attendus.
- Rejeter ou neutraliser les métacaractères SQL.

### 3. Restreindre les privilèges de la base de données
- L’utilisateur SQL de l’application ne doit pas avoir accès à `information_schema`.
- Limiter les droits aux seules tables nécessaires.

### 4. Retirer les données sensibles des métadonnées
Les champs descriptifs ne doivent jamais contenir de secrets, hachages ou indices opérationnels.

### 5. Renforcer le hachage des mots de passe
MD5 ne doit jamais être utilisé à des fins de sécurité. Préférer :
- Argon2
- bcrypt
- PBKDF2

### 6. Améliorer la gestion des erreurs
Les erreurs SQL ne doivent pas être affichées directement à l’utilisateur.

### 7. Mettre en place une surveillance
- Journaliser les requêtes suspectes.
- Détecter l’usage de `UNION` ou d’accès anormaux aux schémas.

---

## Détection et vérification
- **Avant correction :** Les payloads contenant `'`, `UNION SELECT` ou des accès à `information_schema` fonctionnent.
- **Après correction :**
  - Les payloads sont rejetés ou traités comme du texte.
  - Les requêtes SQL ne peuvent plus être altérées.
  - Les champs sensibles ne sont plus accessibles.

---

## Checklist de validation
- [ ] Requêtes préparées utilisées partout.
- [ ] Validation des entrées appliquée.
- [ ] Privilèges SQL restreints.
- [ ] Aucune donnée sensible dans les métadonnées.
- [ ] Suppression totale de MD5.
- [ ] Les injections SQL ne fonctionnent plus.

---

## Recommandations complémentaires
1. Auditer tous les champs exposés aux utilisateurs.
2. Ajouter des scans SQLi automatisés dans la CI.
3. Former les développeurs aux bonnes pratiques SQL sécurisées.
4. Envisager un WAF pour bloquer les patterns d’attaque connus.

---
