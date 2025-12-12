# Darkly — README : Mécanisme de réinitialisation de mot de passe non sécurisé / Champ caché manipulable

## Résumé
Ce document décrit une vulnérabilité dans la fonctionnalité de récupération de mot de passe. Le formulaire envoie une requête POST contenant un champ caché `mail`. Comme cette valeur est entièrement contrôlée côté client et n’est pas validée côté serveur, un attaquant peut modifier l’adresse e‑mail arbitrairement. La soumission de cette valeur modifiée déclenche un comportement serveur non prévu, révélant une flag.

Ce README explique la cause racine, le chemin d’attaque et les stratégies de remédiation appropriées.

---

## Vue d’ensemble de la vulnérabilité
- **Type :** Mécanisme de réinitialisation de mot de passe non sécurisé / Manipulation de paramètre côté client
- **Composant affecté :** Formulaire de récupération de mot de passe
- **Vecteur d’attaque :** Modification du champ caché `mail`
- **Impact :** Comportement de réinitialisation non autorisé et divulgation d’une flag

---

## Preuve de concept (niveau élevé)
1. La page de récupération contient la structure de formulaire suivante :
   ```html
   <form method="POST" action="/lost_password">
     <input type="hidden" name="mail" value="webmaster@borntosec.com" maxlength="15" />
     <button type="submit">Recover password</button>
   </form>
   ```

2. L’application suppose que l’utilisateur ne peut pas modifier le champ caché.

3. À l’aide des outils développeur ou via l’interception de la requête, l’attaquant modifie l’entrée :
   ```html
   <input type="hidden" name="mail" value="attacker@example.com" />
   ```

4. La soumission de la requête amène le backend à traiter cette adresse arbitraire comme légitime.

5. Le serveur retourne alors une réponse contenant la flag.

Cela démontre que le backend fait confiance à des valeurs contrôlées par le client pour une fonctionnalité critique d’authentification.

---

## Analyse de la cause racine
1. **Utilisation de champs cachés pour des paramètres sensibles**
   - Les champs `hidden` sont trivialement modifiables et n’offrent aucune garantie de sécurité.

2. **Absence de validation côté serveur**
   - Le backend ne vérifie pas que l’adresse `mail` correspond à un compte valide ou à l’utilisateur concerné.

3. **Absence de vérification d’identité**
   - Toute valeur d’e‑mail est acceptée sans preuve de possession.

4. **Aucune limitation ni contrôle d’abus**
   - Les endpoints de récupération doivent être protégés contre l’automatisation.

5. **Logique métier défaillante**
   - Le flux de récupération repose sur des hypothèses côté client au lieu de contrôles serveur.

---

## Impact
- **Comportement non autorisé de réinitialisation**
- **Divulgation d’informations sensibles (flag)**
- **Risque de compromission de comptes en environnement réel**

Sévérité : **Élevée**, car les mécanismes d’authentification et d’identification sont compromis.

---

## Remédiations recommandées

### 1. Ne jamais faire confiance aux champs cachés
Les champs cachés ne doivent contenir que des métadonnées d’interface, jamais des identifiants de compte ou des décisions de sécurité.

### 2. Vérifier l’identité côté serveur
Le serveur doit s’assurer que l’e‑mail soumis :
- correspond à un compte existant, et
- est validé via une preuve de possession (challenge).

### 3. Mettre en place un flux de réinitialisation sécurisé
Flux recommandé :
1. L’utilisateur soumet son e‑mail.
2. Si le compte existe, un **token à durée limitée** est envoyé à cet e‑mail.
3. L’utilisateur clique sur le lien contenant le token.
4. Le serveur valide le token et autorise le changement de mot de passe.

### 4. Ne pas divulguer l’existence des comptes
Les messages de réponse doivent être génériques, qu’un e‑mail existe ou non.

### 5. Appliquer une limitation de débit
Limiter les tentatives et bloquer les abus automatisés.

### 6. Auditer et sécuriser tous les formulaires
Aucun formulaire ne doit influencer l’authentification ou l’identité à partir de données client non vérifiées.

---

## Détection et vérification
- **Avant correction :** Modifier le champ caché et soumettre le formulaire permet de déclencher la réponse sensible.
- **Après correction :**
  - Les e‑mails arbitraires n’entraînent aucun comportement sensible.
  - Le flux exige une validation par token.
  - Aucune information sensible n’est divulguée suite à une simple manipulation de formulaire.

---

## Checklist de validation des correctifs
- [ ] Champs cachés retirés de la logique de sécurité.
- [ ] Vérifications serveur de la propriété de l’e‑mail.
- [ ] Workflow de réinitialisation basé sur des tokens.
- [ ] Aucune information sensible dans les réponses.
- [ ] Limitation de débit et journalisation activées.

---

## Recommandations complémentaires
1. Ajouter un CAPTCHA ou une protection anti‑automatisation.
2. Stocker les tokens de réinitialisation sous forme hachée.
3. Auditer tous les endpoints d’authentification pour des failles de logique métier.
4. Réaliser des audits de sécurité réguliers ciblant les flux d’identification.

---
