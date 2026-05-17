# Changelog — drupal-seo

---

## v1.0 — 2026-05-16

**Création initiale — identifié lors de l'audit ultra-critique comme skill manquant**

### Couverture

**`SKILL.md`**
- Quick Decision Table (25+ entrées couvrant Metatag, Sitemap, Pathauto, Redirect, Schema.org, hreflang)
- Anti-patterns critiques (10 entrées)
- Table versioning D8→D11 (Core Web Vitals D11)

**`metatag.md`**
- Installation et modules disponibles
- Configuration globale et par type de contenu
- Tokens Metatag — référence complète
- `hook_metatags_attachments_alter()` pour modifications PHP
- noindex — pages à exclure
- hreflang multilingue
- Canonical et pagination (contenu dupliqué)
- Debug (curl + validateurs externes)

**`sitemap.md`**
- Installation Simple Sitemap
- Configuration par type d'entité (YAML + UI)
- Sitemaps multilingues avec hreflang
- Génération et validation (`drush ssg`)
- Image sitemap pour Google Images
- Exclusions programmatiques
- Cron et génération post-déploiement

**`pathauto.md`**
- Patterns d'alias recommandés par type de contenu
- Options importantes (translitération, longueur max)
- Tokens Pathauto custom (`hook_token_info`, `hook_tokens`)
- Commandes Drush (generate, no-existing)
- Module Redirect — création manuelle et import CSV
- Intégration Pathauto + Redirect (301 automatique)

**`structured-data.md`**
- Pourquoi JSON-LD vs microdata/RDFa
- Configuration via Metatag module
- `hook_page_attachments()` — JSON-LD Article complet avec image et description
- BreadcrumbList dynamique depuis le service breadcrumb Drupal
- Organization globale dans preprocess_html
- FAQ Page Schema.org
- Validation (schema.org/validator, Google Rich Results Test)

**`lessons.md`**
- 8 incidents SEO réels avec corrections détaillées

---

## Compatibilité Drupal

| Skill version | Drupal | Notes |
|--------------|--------|-------|
| v1.0 | D8, D9, D10, D11 | Metatag/Pathauto/Redirect/Simple Sitemap contrib, Core Web Vitals D11 |
