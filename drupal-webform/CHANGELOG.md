# Changelog — drupal-webform

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
