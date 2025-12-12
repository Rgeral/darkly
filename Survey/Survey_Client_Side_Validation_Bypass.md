# Darkly — README : Contournement de la validation côté client dans un formulaire

## Résumé
Ce document décrit une vulnérabilité liée à une validation **exclusivement côté client** dans un formulaire. L’interface propose une liste déroulante avec des valeurs numériques comprises entre 1 et 10. En modifiant la valeur soumise (via les outils développeur du navigateur ou l’interception de la requête), il est possible d’envoyer des valeurs hors plage. La soumission d’une valeur non autorisée déclenche un comportement serveur inattendu et révèle une flag.

Ce write-up explique pourquoi la faille existe, comment elle est exploitée et quelles sont les mesures de remédiation correctes à appliquer côté serveur.

---

## Vue d’ensemble de la vulnérabilité
- **Type :** Contournement de validation côté client / Gestion non sécurisée des entrées
- **Composant affecté :** Élément HTML `<select>` avec des valeurs numériques
- **Impact :** Envoi de valeurs non autorisées menant à la divulgation d’une flag
- **Vecteur d’attaque :** Manipulation des champs de formulaire côté client

---

## Preuve de concept (niveau élevé)
1. L’application fournit une liste déroulante :
   ```html
   <select name="value">
     <option value="1">1</option>
     <option value="2">2</option>
     <option value="3">3</option>
     ...
     <option value="10">10</option>
   </select>
   ```

2. La restriction 1–10 est **uniquement appliquée dans le HTML**, sans contrôle côté serveur.

3. À l’aide des outils développeur, un attaquant peut modifier la valeur avant soumission :
   ```html
   <option value="999">999</option>
   ```
   ou altérer directement le payload de la requête.

4. La soumission d’une valeur supérieure à 10 retourne une réponse cachée contenant la flag.

Cela confirme que le backend fait confiance aux contraintes de l’interface utilisateur au lieu d’appliquer ses propres validations.

---

## Analyse de la cause racine
1. **Dépendance exclusive à la validation côté client**
   - Les contraintes HTML sont triviales à modifier par l’utilisateur.

2. **Absence de validation côté serveur**
   - Le backend accepte la valeur telle quelle, sans vérifier qu’elle est dans la plage autorisée.

3. **Confiance implicite dans l’UI**
   - Le serveur suppose que le client ne peut pas modifier les champs du formulaire.

4. **Chemins de code déclenchés par des valeurs invalides**
   - Les entrées hors plage activent une logique alternative exposant des informations internes.

---

## Impact
- **Accès non autorisé à des fonctionnalités restreintes**
- **Divulgation de données** (flag)
- **Risque d’abus de logique métier** dans des scénarios réels (élévation de privilèges, actions non prévues)

Sévérité estimée : **Moyenne à Élevée**, selon la nature des actions déclenchées par des valeurs invalides.

---

## Remédiations recommandées

### 1. Appliquer une validation côté serveur
Le serveur doit vérifier explicitement la plage autorisée :
```
if value < 1 or value > 10:
    rejeter_la_requete()
```
Les contrôles serveur sont **autoritatifs**.

### 2. Considérer toute entrée client comme non fiable
- Valider systématiquement le type, la plage et le format de chaque paramètre.
- Ne jamais se fier aux contraintes HTML pour la sécurité.

### 3. Éviter d’exposer une logique sensible basée sur des valeurs brutes
- Ne pas révéler de flag ou d’informations sensibles en fonction d’un simple paramètre numérique.
- Mettre en place authentification et contrôles d’accès appropriés.

### 4. Dupliquer les restrictions côté client uniquement pour l’UX
- Les validations client améliorent l’expérience utilisateur, mais ne remplacent jamais les contrôles serveur.

### 5. Journaliser les entrées anormales
- Enregistrer les valeurs hors plage pour détecter des tentatives de manipulation.

---

## Détection et vérification
- **Avant correction :** Modifier la valeur soumise (>10) permet d’obtenir la flag.
- **Après correction :**
  - Les valeurs hors plage sont rejetées (ex. HTTP 400).
  - Aucune information sensible n’est divulguée.
  - Le comportement serveur reste identique quelle que soit la manipulation client.

---

## Checklist de validation
- [ ] Validation de plage implémentée côté serveur.
- [ ] Les contraintes UI reflètent les règles serveur sans les remplacer.
- [ ] Les valeurs invalides ne déclenchent aucune réponse sensible.
- [ ] Journalisation activée pour les entrées anormales.

---

## Recommandations complémentaires
1. Appliquer une validation de schéma stricte sur toutes les entrées.
2. Utiliser des frameworks offrant la validation automatique des requêtes.
3. Revoir régulièrement le code de traitement des formulaires.
4. Ajouter des tests automatisés pour les cas d’entrées invalides.

---
