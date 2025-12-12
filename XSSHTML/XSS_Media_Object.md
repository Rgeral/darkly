# Darkly — README : XSS via inclusion de ressource dans `page=media` (paramètre `src`)

## Résumé
Ce document décrit une vulnérabilité de **Cross-Site Scripting (XSS)** présente sur la page  
`index.php?page=media&src=...`.  
La valeur du paramètre `src` est injectée directement dans l’attribut `data` d’un élément `<object>`. En fournissant des URI contrôlées (notamment des `data:` URI contenant du HTML encodé en base64), le navigateur charge et interprète le contenu fourni, permettant l’exécution de JavaScript dans le contexte du site.

---

## Contexte
Sur la page `media`, le HTML rendu contient un élément de la forme :
```html
<object data="http://192.168.64.3/images/nsa_prism.jpg"></object>
```
L’application construit cet attribut à partir de la valeur fournie par l’utilisateur via `src`, sans validation suffisante.

---

## Découverte de la faille
1. Inspection du HTML généré pour `?page=media&src=nsa`.
2. Hypothèse : si `src` est utilisé tel quel pour construire `data`, alors des URI arbitraires (`data:`, `javascript:`, etc.) peuvent être injectées.
3. Test pratique concluant avec une `data:` URI.

---

## Preuve de concept (PoC)

### Exemple minimal
```bash
http://192.168.64.3/index.php?page=media&src=data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==
```

### Reproduction via `curl`
```bash
curl -s "http://192.168.64.3/index.php?page=media&src=data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==" | sed -n '1,200p'
```

L’URI `data:` contient un document HTML arbitraire, interprété par le navigateur, ce qui permet l’exécution de JavaScript.

---

## Analyse technique

- **Type :** XSS via inclusion de ressource (`data:` URI) dans `<object>`
- **Vecteur :** paramètre `src` non validé
- **Cause racine :**
  - absence de whitelist côté serveur,
  - acceptation de schémas d’URI dangereux,
  - utilisation de `<object>` pour du contenu contrôlé par l’utilisateur.

Pourquoi la flag peut apparaître même sans payload complet : la logique serveur semble détecter certains schémas (`data:`) ou préfixes et déclencher un comportement spécifique, sans valider finement le contenu.

---

## Impact
- **Exécution de JavaScript arbitraire** dans le contexte du domaine
- **Vol de cookies / sessions**
- **Actions au nom de l’utilisateur**
- **Exfiltration d’informations sensibles**

Sévérité estimée : **Élevée**.

---

## Recommandations prioritaires

### 1) Whitelist / mapping côté serveur (obligatoire)
N’accepter que des identifiants connus et les mapper vers des ressources internes sûres.
```php
$allowed = ['nsa','cia'];
$src = $_GET['src'] ?? '';
if (!in_array($src, $allowed, true)) {
  header('HTTP/1.1 404 Not Found'); exit;
}
$imgPath = '/images/' . basename($src) . '_prism.jpg';
```

### 2) Refuser explicitement les schémas dangereux
```php
if (preg_match('#^[a-zA-Z][a-zA-Z0-9+\-.]*:#', $src)) {
  header('HTTP/1.1 400 Bad Request'); exit;
}
```

### 3) Éviter `<object>` pour du contenu utilisateur
Utiliser `<img>` avec des sources internes contrôlées. `<object>` peut charger des documents HTML/JS interprétables.

### 4) Déployer des en-têtes de sécurité
```
Content-Security-Policy: default-src 'self'; img-src 'self'; object-src 'none'; script-src 'self';
X-Content-Type-Options: nosniff
```

### 5) Échapper toutes les sorties
```php
htmlspecialchars($value, ENT_QUOTES, 'UTF-8');
```

---

## Patch minimal suggéré (extrait PHP)
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

## Checklist de validation
- [ ] `src` validé par whitelist
- [ ] Schémas `data:`, `javascript:` rejetés
- [ ] `<object>` supprimé pour le contenu dynamique
- [ ] CSP active avec `object-src 'none'`
- [ ] Échappement systématique des sorties

---

## Conclusion
La faille provient de l’utilisation directe d’un paramètre utilisateur pour remplir l’attribut `data` d’un `<object>`.  
La correction repose sur une **validation stricte côté serveur**, l’**interdiction des schémas dangereux**, l’**abandon de `<object>`** pour le contenu utilisateur et l’**application d’en-têtes de sécurité**. Après ces mesures, l’attaque via `data:` est neutralisée.
