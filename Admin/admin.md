# Darkly — README : Exposition de fichiers sensibles via `robots.txt` et stockage faible des identifiants

## Résumé
Ce document analyse une chaîne de vulnérabilités dans laquelle des informations d’authentification sensibles sont exposées indirectement via le fichier `robots.txt`. Des directives d’exclusion pour les moteurs de recherche pointent vers un répertoire caché, lequel contient un fichier `.htpasswd` stockant un mot de passe haché en MD5. Une fois le hash cassé, il permet un accès non autorisé à la page `/admin`, révélant finalement un flag.

L’objectif de ce README est d’expliquer clairement comment la vulnérabilité apparaît, comment elle peut être reproduite, et comment elle doit être corrigée afin d’éviter des problèmes similaires.

---

## Vue d’ensemble de la vulnérabilité
- **Type :** Divulgation d’information, stockage faible des identifiants, contrôle d’accès insuffisant
- **Point d’entrée :** `robots.txt`
- **Chemins affectés :** `/robots.txt`, `/whatever`, `/admin`, `.htpasswd`
- **Problème sous-jacent :** Des ressources sensibles sont exposées via des emplacements prévisibles et les identifiants sont stockés à l’aide d’un algorithme de hachage cryptographiquement faible.

---

## Preuve de concept (niveau élevé)
1. Accéder à :
   ```
   http://10.14.200.249/robots.txt
   ```
   Le fichier contient la directive :
   ```
   Disallow: /whatever
   ```

2. La consultation de ce répertoire révèle un fichier `.htpasswd` contenant :
   ```
   root:437394baff5aa33daa618be47b75cb49
   ```

3. Le hash utilise **MD5**, une fonction de hachage obsolète et facilement cassable.
   Une fois cassé, il fournit un mot de passe valide.

4. En utilisant ce mot de passe sur :
   ```
   http://10.14.200.249/admin
   ```
   on obtient un accès au panneau d’administration, qui expose un flag.

Cette chaîne montre comment des métadonnées apparemment anodines (`robots.txt`) peuvent révéler des ressources sensibles lorsqu’elles sont mal conçues ou mal protégées.

---

## Analyse de la cause racine
1. **Utilisation inappropriée de `robots.txt` pour cacher du contenu**
   - Le fichier `robots.txt` est destiné aux robots d’indexation, pas à la sécurité.
   - Les répertoires listés dans `Disallow` sont visibles publiquement et attirent souvent les attaquants.

2. **Exposition de fichiers liés à l’authentification**
   - Le fichier `.htpasswd` est accessible depuis la racine web.
   - Aucun contrôle d’accès n’empêche son téléchargement direct.

3. **Hachage cryptographique faible (MD5)**
   - MD5 est vulnérable aux attaques par force brute et est largement considéré comme non sécurisé.
   - Stocker des mots de passe en MD5 permet un cassage hors-ligne trivial.

4. **Absence de protection côté serveur pour les répertoires sensibles**
   - Les zones sensibles sous `/whatever` ne disposent pas de règles de configuration telles que `Deny from all`, d’exigences d’authentification ou d’un retrait de la racine publique.

5. **Absence de limitation de débit et de surveillance**
   - L’accès au fichier `.htpasswd` n’est pas détecté.
   - Les schémas d’attaque issus de tentatives d’énumération ne sont pas identifiés.

---

## Impact
- **Accès administrateur non autorisé :** Les identifiants cassés permettent la connexion à l’interface admin.
- **Atteinte à la confidentialité :** L’exposition de mots de passe hachés compromet le système d’authentification.
- **Surface d’attaque chaînable :** Avec des identifiants administrateur, un attaquant peut modifier le contenu, élever ses privilèges ou injecter du code malveillant.
- **Faux sentiment de sécurité dû à `robots.txt` :** Les développeurs peuvent croire à tort que des chemins « cachés » sont protégés.

Évaluation de la sévérité : **Élevée**, en raison de l’accès administratif direct résultant de données d’authentification accessibles publiquement.

---

## Remédiation (correctifs recommandés)
Mettre en œuvre les mesures suivantes pour éliminer la vulnérabilité et éviter toute récidive.

### 1. Retirer les répertoires sensibles de la racine web publique
- Les fichiers d’authentification comme `.htpasswd` ne doivent jamais être accessibles directement.
- Ils doivent être stockés hors de la racine publique ou protégés par des règles d’accès strictes.

### 2. Ne pas s’appuyer sur `robots.txt` pour la sécurité
- `robots.txt` ne fait que communiquer des préférences d’indexation ; il n’offre aucune protection.
- Éviter totalement d’y lister des répertoires sensibles.
- Si un répertoire ne doit pas être indexé, cela doit être combiné à de véritables restrictions d’accès.

### 3. Appliquer un contrôle d’accès strict côté serveur
- Mettre en place des restrictions au niveau des répertoires via la configuration serveur (ex. `.htaccess`, règles Nginx).
- Interdire l’accès direct aux fichiers de configuration et d’authentification.

### 4. Utiliser un hachage robuste pour les identifiants
- Remplacer MD5 par des algorithmes de dérivation de clés lents et salés, tels que :
  - `bcrypt`
  - `Argon2`
  - `PBKDF2`
- Faire une rotation de tous les identifiants exposés.

### 5. Renforcer l’interface d’administration
- Ajouter une limitation de tentatives, imposer des mots de passe forts et des seuils de blocage.
- Envisager une authentification à deux facteurs pour les interfaces sensibles.

### 6. Améliorer la journalisation et la surveillance
- Journaliser les tentatives de connexion échouées.
- Détecter les schémas d’énumération suspects.
- Déclencher des alertes lors d’accès anormaux à des chemins sensibles.

---

## Modèles de configuration sécurisée (exemples)

### Déplacer `.htpasswd` hors de la racine web
```
/var/www/darkly/               (public)
/etc/darkly/auth/.htpasswd     (privé)
```
La configuration du serveur web doit référencer explicitement le chemin privé.

### Interdire l’accès direct via des règles serveur (exemple Apache)
```
<FilesMatch "^(\.htpasswd|\.htaccess)$">
  Require all denied
</FilesMatch>
```

### Exemple de hachage robuste (pseudo-code)
```
stored_hash = bcrypt.hash(password_input)
verify(password_input, stored_hash)
```

---

## Détection et vérification
- **Avant correctif :** Vérifier l’accessibilité de `/whatever` et de `.htpasswd` via des URLs directes.
- **Après correctif :** Toute tentative d’accès doit retourner `403 Forbidden` ou `404 Not Found`.
- **Audit des identifiants :** Aucun mot de passe haché en MD5 ne doit subsister.
- **Test du panneau admin :** La page `/admin` doit exiger des identifiants valides et ne pas être accessible via des mots de passe divulgués.

---

## Checklist de validation des correctifs
- [ ] `.htpasswd` supprimé de la racine publique.
- [ ] Accès direct aux fichiers sensibles interdit.
- [ ] Hachage des mots de passe admin migré vers un KDF moderne.
- [ ] `robots.txt` n’expose plus de répertoires sensibles.
- [ ] Journalisation et alertes activées.
- [ ] Test de pénétration de revalidation effectué.

---

## Recommandations supplémentaires
1. Réaliser un audit complet du système de fichiers pour vérifier qu’aucun autre fichier sensible n’est accessible via le web.
2. Mettre en place une rotation régulière des mots de passe et des règles de complexité renforcées.
3. Ajouter des outils d’analyse automatisée pour détecter les hachages faibles ou les fichiers exposés.
4. Effectuer des revues de sécurité périodiques après chaque évolution fonctionnelle.
