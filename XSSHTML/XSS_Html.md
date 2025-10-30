# Rapport — XSS sur `page=media` (param `src`)

## Contexte
Sur la page `index.php?page=media&src=...`, la valeur du paramètre `src` est insérée dans un élément `<object data="...">`.  
En fournissant des URIs contrôlées (notamment des `data:` URI contenant du HTML encodé en base64), le navigateur charge et interprète le contenu fourni — ce qui permet d’exécuter du JavaScript dans le même contexte que le site.

---

## Comment on a trouvé la faille
1. Inspection du HTML rendu pour `?page=media&src=nsa` :
```html
<object data="http://192.168.64.3/images/nsa_prism.jpg"></object>
```
2. Hypothèse : si `src` est utilisée telle quelle pour construire `data`, alors on peut fournir un URI arbitraire (`data:`, `javascript:`, …).  
3. Test pratique (PoC) : appeler l’URL suivante dans un navigateur ou avec `curl` :
```bash
http://192.168.64.3/index.php?page=media&src=data:text/html;base64,PHNjcmlwdD5hbGVydCg==
```
Avec ceci, on active le flag en envoyant au serveur du code malicieux

---

## Description précise de la faille
- **Type** : Cross-Site Scripting (XSS) via inclusion de ressource (`data:` URI) dans `<object>`.  
- **Vecteur** : paramètre `src` non validé, inséré directement dans l'attribut `data` d’un `<object>`, permettant de charger un document HTML arbitraire.  
- **Impact** : exécution de JavaScript dans le contexte du domaine (vol de cookies, actions au nom de l’utilisateur, exfiltration d’informations sensibles).

Pourquoi la flag peut apparaître sans payload complet : le code serveur détecte probablement la présence d’un schéma (`data:`) ou d’un préfixe et déclenche une logique (inclusion d’un fragment, affichage d’un contenu spécial, backdoor pédagogique). Autrement dit, le serveur se contente souvent d’identifier le schéma plutôt que de valider le contenu complet.

---

## Preuve (PoC)
Reproduire le test qui montre la vulnérabilité :
```bash
# data: URI contenant <script>alert(1)</script> en base64
curl -s "http://192.168.64.3/index.php?page=media&src=data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==" | sed -n '1,200p'
```
Ou ouvrir l’URL dans un navigateur pour voir l’exécution/affichage du contenu injecté.

---

## Pourquoi c’est dangereux (en une phrase)
Permettre à un utilisateur de contrôler un attribut `data` d’un `<object>` autorise le chargement et l’exécution de documents HTML/JS arbitraires dans le contexte du site — c’est l’équivalent d’un backdoor pour l’exécution de code côté client.

---

## Recommandations (ce qu’il faut faire — prioritaire)
- **Whitelist / mapping côté serveur (obligatoire)**  
  N’accepter que des identifiants connus (ex : `nsa`, `cia`) et mapper ces identifiants sur des fichiers internes sûrs ; ne jamais construire un URI ou chemin directement avec une valeur fournie par l’utilisateur.
  ```php
  $allowed = ['nsa','cia'];
  $src = $_GET['src'] ?? '';
  if (!in_array($src, $allowed, true)) {
    header('HTTP/1.1 404 Not Found'); exit;
  }
  $imgPath = '/images/' . basename($src) . '_prism.jpg';
  ```

- **Refuser explicitement les schémas dangereux**  
  Rejeter toute valeur contenant un schéma (`data:`, `javascript:`, `file:`...). Exemple :
  ```php
  if (preg_match('#^[a-zA-Z][a-zA-Z0-9+\-.]*:#', $src)) {
    header('HTTP/1.1 400 Bad Request'); exit;
  }
  ```

- **Ne pas utiliser `<object>` pour du contenu utilisateur**  
  Pour afficher des images, utiliser `<img src="...">` avec des sources internes contrôlées ; `<object>` peut charger des documents interprétables (HTML/JS) et doit être évité pour du contenu dynamique.

- **Déployer une CSP stricte et headers MIME**  
  Exemple d’en-tête à ajouter :
  ```
  Content-Security-Policy: default-src 'self'; img-src 'self'; object-src 'none'; script-src 'self';
  X-Content-Type-Options: nosniff
  ```

- **Échapper toute sortie**  
  Lorsque tu affiches une valeur (ex. `File: ...`), utilise `htmlspecialchars($value, ENT_QUOTES, 'UTF-8')`.

- **Ne jamais inclure/exécuter de fragments basés sur l’entrée utilisateur**  
  Interdire tout `include()` dynamique fondé sur un paramètre GET/POST sans validation/whitelist.

---

## Patch minimal suggéré (extrait PHP)
Remplacer la construction directe d’un `<object>` par un mapping sûr + `<img>` :
```php
$allowed = ['nsa','cia'];
$src = $_GET['src'] ?? '';
if (!in_array($src, $allowed, true)) {
  header('HTTP/1.1 404 Not Found');
  exit;
}
$imgPath = '/images/' . basename($src) . '_prism.jpg';
echo '<div class="media">';
echo '<p>File: ' . htmlspecialchars($src . '_prism.jpg', ENT_QUOTES, 'UTF-8') . '</p>';
echo '<img src="' . htmlspecialchars($imgPath, ENT_QUOTES, 'UTF-8') . '" alt="">';
echo '</div>';
```

---

## Conclusion
La faille provient d’une **utilisation non validée** d’un paramètre utilisateur pour remplir un attribut `data` d’un `<object>`. En pratique, corriger nécessite d’imposer une whitelist/mapping côté serveur, d’interdire les schémas dangereux, d’éviter `<object>` pour du contenu utilisateur, et d’ajouter des en-têtes de sécurité (CSP, nosniff) ainsi que l’échappement systématique des sorties. Après ces changements l’attaque via `data:` sera neutralisée et l’application sera beaucoup plus robuste.

---
