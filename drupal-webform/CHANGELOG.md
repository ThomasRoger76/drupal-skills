# Changelog — drupal-webform

---

## v1.1 — 2026-06-08

**Mise à jour Drupal 11 currency + complétude plugins custom**

### Corrigé
- **`webform-handlers.md`** — Handler custom migré de l'annotation Doctrine `@WebformHandler` (dépréciée) vers l'attribut PHP `#[WebformHandler]` (Webform 6.2+ / D11). Ajout de `declare(strict_types=1)`, des `use` complets et de `new TranslatableMarkup()`.
- **`webform-handlers.md`** — Injection du `http_client` via `create()` + propriété typée au lieu de `\Drupal::service()` dans `postSave()`. `true` → `TRUE`.
- **`webform-elements.md`** — Correction du type CAPTCHA : `drupal/recaptcha` fournit la v2, pas la v3 (qui nécessite `drupal/recaptcha_v3`). Commandes `drush en` préfixées Docker natif.
- **`webform-api.md`** — Retrait de `hal` (déprécié/retiré en D10+) de la commande d'activation REST.

### Ajouté
- **`webform-elements.md`** — Section « Élément custom — `#[WebformElement]` » : plugin complet (champ SIRET + validation Luhn serveur via `#element_validate`), avec rappel config-avant-code (composites UI d'abord).
- **`SKILL.md`** — Convention Drush Docker natif. Lignes versioning : attribut PHP `#[WebformHandler]`/`#[WebformElement]` (D10.2 6.2+ / D11) vs annotation dépréciée.
- **`lessons.md`** — 2 incidents : annotation dépréciée en D11, `\Drupal::service()` non testable dans un plugin.

---

## v1.0 — 2026-05-16

**Création initiale — identifié lors de l'audit ultra-critique**

### Couverture

**`SKILL.md`**
- Quick Decision Table (25+ entrées couvrant éléments, handlers, soumissions, accès, API)
- Anti-patterns critiques (8 entrées)
- Table versioning D8→D11

**`webform-elements.md`**
- Catalogue complet des types d'éléments (50+ types en catégories)
- Configuration YAML complète d'un formulaire
- Logique conditionnelle `#states` (visible, required, AND/OR)
- Wizard multi-étapes avec `webform_wizard_page`
- Pré-remplissage (tokens, URL query, programmatique)
- Éléments Computed avec Twig
- CAPTCHA et Honeypot anti-spam

**`webform-handlers.md`**
- Handler Email — configuration complète (YAML + options)
- Handler Remote POST — JSON payload avec tokens
- Handler Custom `WebformHandlerBase` — classe complète avec CRM sync
- Mode Test pour valider sans envoyer

**`webform-submissions.md`**
- `WebformSubmission::load()` et chargement multiple
- Création programmatique avec données custom
- Modification de soumissions existantes
- `hook_webform_submission_presave/insert()`
- `hook_webform_submission_form_alter()` pour modifier le rendu
- Rétention des données et purge RGPD
- Export CSV

**`lessons.md`**
- 7 incidents Webform réels avec corrections

---

## Compatibilité Drupal

| Skill version | Drupal | Notes |
|--------------|--------|-------|
| v1.0 | D8, D9, D10, D11 | Webform contrib — toutes versions |
| v1.1 | D8 → D11 | Plugins en attribut PHP par défaut (D10.2+/D11, Webform 6.2+) |
