# Changelog — drupal-seo

---

## v1.1 — 2026-06-09

**Passe de fiabilité — correction de défauts factuels et alignement aux standards Docker natif**

### Corrigé
- **`sitemap.md`** : nom de hook erroné `hook_simple_sitemap_links_alter_all` → `hook_simple_sitemap_links_alter` (signature `$sitemap` corrigée).
- **`redirects.md`** : section « Intégration Pathauto » référençait par erreur « Webform → Settings » → corrigé en `/admin/config/search/path/settings` avec le libellé exact d'« Update action ».
- **`pathauto.md`** : libellé d'option Pathauto aligné sur le wording réel de l'UI + mention de la dépendance `drupal/redirect`.
- **`metatag.md`** : tokens OG/Twitter image corrigés pour les champs **Media** (`…:entity:field_media_image:entity:url`) avec note sur le cas File vs Media (cohérence avec lessons.md).
- **`structured-data.md`** : ajout des cache metadata obligatoires (`#cache['tags']` node:ID, `#cache['contexts']` url.path) pour éviter le JSON-LD obsolète servi depuis le cache ; garde-fou `isAdminRoute()` sur l'Organization pour ne pas polluer l'admin.
- **`SKILL.md`** : lien mort `agents/seo-audit.md` supprimé ; renvois Core Web Vitals corrigés (pointaient vers `sitemap.md` au lieu du skill `drupal-performance`).

### Ajouté
- **`SKILL.md`** : convention d'exécution **Docker natif** (`docker compose exec php drush …`, jamais `ddev`) conforme aux standards projet.
- **`lessons.md`** : leçon « JSON-LD obsolète servi depuis le cache » + section de corrections de cohérence.

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
