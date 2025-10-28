# Faille *redirect* — explication détaillée, PoC et mitigation

> Document rédigé pour le rapport CTF. Contenu en français.

---

## 1) Contexte & comment nous avons trouvé la faille

**PoC reproductible (ce que nous avons exécuté)** :

```bash
curl -v "http://192.168.64.3/index.php?page=redirect&site=http://example.com"
```

- Le serveur répond en `200 OK` et dans le corps HTML on voit directement :

```
Good Job Here is the flag : b9e775a0291fed784a2d9680fcfad7edd6b8cdf87648da647aaf4bba288bcab3
```

- Dans les en-têtes de réponse nous avons aussi observé :



**Observation** : la route `page=redirect` avec un paramètre `site=` renvoie du contenu sensible (ici un flag). L’accès à ce contenu ne nécessite pas d’authentification forte — il suffit d’appeler l’URL.

**Comment nous l’avons trouvé** :
- Inspection du code HTML et des commentaires sur la page principale (indices `You must come from : "https://www.nsa.gov/"` et `ft_bornToSec`).
- Exploration manuelle des paramètres visibles (`?page=survey`, `?page=member`, `?page=redirect`).
- Test avec `curl -v` pour obtenir en-têtes et corps ; l’option `-v` révèle `Set-Cookie` et facilite la relecture.

---

## 2) Qu’est-ce qu’une faille *Redirect* ?

### Définition
Une **open redirect** (redirection ouverte) est une vulnérabilité où une application web accepte une URL de destination fournie par l’utilisateur et redirige vers cette URL **sans** validation suffisante. Concrètement :
- un paramètre (ex : `?next=...`, `?url=...`, `?site=...`) est utilisé pour construire une redirection ;
- l’application redirige vers la valeur fournie par l’utilisateur sans vérifier qu’elle est autorisée.

### Variantes et comportements observés
- **Open Redirect classique** : l’URL fournie est utilisée telle quelle pour `Location:` (HTTP 3xx) ou pour un `window.location` côté client.
- **Parameter-triggered content** : au lieu d’effectuer une redirection, l’application peut exécuter une logique différente (afficher une page spéciale, révéler des données) lorsque le paramètre contient une certaine valeur. C’est ce que nous avons vu : le paramètre `site` déclenche l’affichage d’un flag.
- **Redirect + token leak** : redirection vers un domaine externe peut servir à exfiltrer tokens, cookies mal configurés ou mener des attaques de phishing.

### Pourquoi c’est dangereux
- **Phishing** : un attaquant peut construire un lien vers le site vulnérable, puis rediriger la victime vers un site malveillant en conservant le domaine légitime dans le lien (confiance trompeuse).
- **Exfiltration de données** : si la page redirige et inclut ou transmet des tokens/session IDs, ceux-ci peuvent être capturés.
- **Bypass d’ACL** : si des fonctionnalités sensibles sont déclenchées par des paramètres contrôlables, un attaquant peut accéder à des pages réservées ou du contenu non prévu.

### Preuves (observées dans ce cas)
- Le paramètre `site` a permis d’obtenir le flag sans authentification.


---

## 3) Comment expliquer techniquement (flow simplifié)

1. Client → Serveur : `GET /index.php?page=redirect&site=http://example.com`
2. Serveur : lit `$_GET['page'] == 'redirect'` et potentiellement `$_GET['site']`.
3. Logic : si la page correspond à `redirect`, alors afficher/renvoyer un contenu spécial (ici flag) ou effectuer une redirection. Il n’y a pas de validation/authentification stricte.
4. Serveur renvoie `200 OK` avec le contenu et `Set-Cookie`.

La vulnérabilité provient donc d’une **logique côté serveur** qui expose une action sensible (redirection ou affichage) sur la base d’un paramètre contrôlé par l’utilisateur.

---

## 4) Exemple de code vulnérable (PHP) et pourquoi il l’est

### Exemple vulnérable :

```php
// index.php (extrait)
$page = $_GET['page'];
$site = isset($_GET['site']) ? $_GET['site'] : '';
if ($page === 'redirect') {
    // vulnérable : aucune validation de $site
    header('Location: ' . $site);
    exit;
}
```

**Problèmes** :
- `header('Location: ' . $site)` envoie l’utilisateur vers n’importe quelle URL fournie ;
- pas de whitelist, pas de validation, pas de nettoyage ;
- si le code au lieu de rediriger exécute une logique conditionnelle (affiche du contenu) selon la présence du paramètre, alors il suffit d’appeler l’URL pour obtenir le contenu.

---

## 5) Mitigations — comment se protéger (pratiques et exemples)

### Principes généraux
1. **Ne jamais faire confiance aux paramètres fournis par l’utilisateur** — surtout lorsqu’ils contrôlent des redirections ou l’accès à des ressources.
2. **Utiliser une whitelist** d’URLs/chemins autorisés plutôt qu’une blacklist.
3. **Pour les redirections internes** : préférer des clés internes (tokens, alias) plutôt que des URLs complètes exposées.
4. **Protéger les cookies** (`HttpOnly`, `Secure`, `SameSite`) et ne pas utiliser des cookies prévisibles pour l’élévation de privilèges.
5. **Éviter d’accorder des droits** en se basant sur `Referer` ou `User-Agent` — ces en-têtes sont contrôlables par le client.

### Exemple sécurisé en PHP (whitelist)

```php
// whitelist de destinations autorisées (internes ou domaines de confiance)
$whitelist = [
    'https://example.com',
    'https://www.example.com',
    '/dashboard',
    '/profile'
];

$site = $_GET['site'] ?? '';

// Normaliser la valeur (par ex. convertir les chemins relatifs)
$parsed = parse_url($site);
$normalized = '';
if ($parsed === false) {
    // valeur non parsable -> refuser
    http_response_code(400);
    exit('Bad site');
}

if (isset($parsed['host'])) {
    $normalized = $parsed['scheme'] . '://' . $parsed['host'];
    if (isset($parsed['path'])) $normalized .= $parsed['path'];
} else {
    // chemin relatif
    $normalized = $site;
}

if (!in_array($normalized, $whitelist, true)) {
    // redirection par défaut sûre
    header('Location: /');
    exit;
}

header('Location: ' . $normalized);
exit;
```

### Variante recommandée : utiliser des tokens/aliases
Au lieu de `?site=https://evil`, on peut utiliser `?to=dashboard` et résoudre `dashboard` côté serveur :

```php
$destinations = [
  'dashboard' => '/dashboard',
  'profile' => '/user/profile'
];
$to = $_GET['to'] ?? '';
if (!array_key_exists($to, $destinations)) {
  header('Location: /');
  exit;
}
header('Location: ' . $destinations[$to]);
exit;
```

### Headers & cookies
- `Set-Cookie` : ajouter `HttpOnly; Secure; SameSite=Strict` pour éviter l’exfiltration côté client.
- Ne pas considérer `Referer` ou `User-Agent` comme preuve d’authenticité : ils sont falsifiables.

### Logging & monitoring
- Logger les redirections effectuées (source IP, paramètre fourni).
- Déclencher des alertes si des redirections vers domaines externes apparaissent fréquemment.

---

## 6) Checklist pour l’auditeur / correcteur
- [ ] Le paramètre utilisé pour rediriger est-il validé ? (whitelist)
- [ ] L’application autorise-t-elle les redirections vers des domaines externes ?
- [ ] Les cookies sensibles sont-ils `HttpOnly` et `Secure` ?
- [ ] Le code n’accorde-t-il pas d’accès sur simple présence d’un paramètre ? (feature-flag non authentifié)
- [ ] Les en-têtes `Referer`/`User-Agent` sont-ils utilisés pour des décisions de sécurité ? Si oui, corriger.

---

## 7) Exemple de texte court à coller dans un rapport (résumé)
> **Résumé** : l’endpoint `index.php?page=redirect&site=...` accepte une URL de destination contrôlée par l’utilisateur et exécute une redirection / action sans validation. En CTF cette route a révélé un flag; en production, une telle vulnérabilité permettrait des attaques de phishing, exfiltration de tokens, et bypasss de contrôles d’accès. Recommandation : appliquer une whitelist, utiliser des alias internes pour redirections, protéger les cookies, et ne pas se baser sur `Referer`/`User-Agent` pour la sécurité.

---

## 8) Références utiles (lecture)
- OWASP — Unvalidated Redirects and Forwards: https://owasp.org/www-project-top-ten/ (chercher "Unvalidated Redirects and Forwards")
- Bonnes pratiques cookies: MDN Web Docs

---

> Si tu veux, je peux générer aussi un fichier `poC.sh` (script bash) qui automatise la preuve (curl + extraction du flag + sauvegarde du cookie) ou préparer une version courte (une page) à rendre au professeur.

