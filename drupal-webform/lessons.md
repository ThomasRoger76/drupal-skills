# Leçons — drupal-webform

Problèmes Webform rencontrés en projets Drupal réels.

---

## Comment ajouter une leçon

Après chaque incident Webform :
1. Identifier si le skill aurait pu prévenir l'erreur
2. Ajouter une entrée avec symptôme, cause, correction, prévention
3. Ajouter une ligne dans CHANGELOG.md

---

### 2026-05-16 — Emails de confirmation jamais reçus — handler Email mal configuré

- **Symptôme :** Les utilisateurs ne reçoivent pas l'email de confirmation après soumission
- **Cause :** Le handler Email était configuré avec `To: admin@site.com` au lieu de `To: [webform_submission:values:email]`
- **Correct :** Handler Email → To field → `[webform_submission:values:CHAMP_EMAIL]`
- **Prévention :** Toujours tester les handlers en mode Test (Settings → Test) avant la mise en production. Vérifier dans Maildev.

### 2026-05-16 — Soumissions non stockées — perte de données

- **Symptôme :** Les soumissions ne sont pas visibles dans Results → Submissions
- **Cause :** Settings → Submissions → "Store submissions" n'était pas activé
- **Correct :** Activer "Store submissions" dans les Settings du formulaire. Les soumissions futures seront stockées.
- **Prévention :** Toujours activer "Store submissions" sur les formulaires importants. Les emails peuvent échouer — la DB est le filet de sécurité.

### 2026-05-16 — Remote POST silencieusement en échec — données CRM perdues

- **Symptôme :** Les formulaires sont soumis correctement mais les leads n'arrivent pas dans le CRM
- **Cause :** Le handler Remote POST échouait (API CRM down) mais sans log visible
- **Correct :** Activer le logging des handlers : `debug: true` dans la config du handler. Vérifier `/admin/reports/dblog`
- **Prévention :** Ajouter un handler Email en backup. Monitorer les logs watchdog. Toujours avoir "Store submissions" activé.

### 2026-05-16 — SPAM massif — formulaire public sans protection

- **Symptôme :** 500 soumissions spam en une nuit sur un formulaire de contact
- **Cause :** Aucune protection anti-spam sur un formulaire accessible anonymement
- **Correct :** Activer Honeypot (Settings → Spam protection) ou reCAPTCHA v3
- **Prévention :** Tout formulaire public doit avoir au minimum Honeypot. reCAPTCHA pour les formulaires sensibles.

### 2026-05-16 — Pré-remplissage XSS — injection via URL query

- **Symptôme :** Un attaquant injecte `?message=<script>alert(1)</script>` dans l'URL et le formulaire exécute le script
- **Cause :** Pré-remplissage depuis `$_GET` sans sanitisation dans un hook personnalisé
- **Correct :** Utiliser les tokens Webform sécurisés ou `Html::escape()` + `$request->query->get()` (retourne une string sanitisée)
- **Prévention :** Ne jamais utiliser `$_GET` directement. Utiliser `\Drupal::request()->query->get('champ')` et les tokens Webform.

### 2026-05-16 — Wizard cassé après ajout d'un élément — ordre invalide

- **Symptôme :** Après ajout d'un élément, les étapes du wizard s'affichent dans le mauvais ordre
- **Cause :** L'élément a été ajouté en dehors d'une `webform_wizard_page` — il apparaît sur toutes les étapes
- **Correct :** Dans l'éditeur YAML, déplacer l'élément à l'intérieur de la bonne `webform_wizard_page`
- **Prévention :** En mode wizard, toujours vérifier dans le YAML que chaque élément est bien indented sous sa page

### 2026-05-16 — Conditions `#states` non déclenchées — sélecteur jQuery incorrect

- **Symptôme :** Le champ conditionnel reste toujours visible/caché quelle que soit la valeur du champ parent
- **Cause :** Le sélecteur jQuery dans `#states` ne correspond pas au nom HTML du champ (Webform ajoute un préfixe)
- **Correct :** Inspecter l'attribut `name` du champ HTML avec les DevTools → utiliser exactement `:input[name="EXACT_NAME"]`
- **Prévention :** Tester les `#states` en mode preview avant d'activer le formulaire
