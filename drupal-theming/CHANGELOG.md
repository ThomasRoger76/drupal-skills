# Changelog — drupal-theming

---

## v1.2 — 2026-05-16

**Nouvel agent :**
- `agents/theme-generator.md` — génère un thème Drupal 11 complet from scratch (`.info.yml`, libraries, preprocess, regions, SDC, build pipeline Vite). Supporte `--type=standalone|bootstrap5|sdc`. Inclut les règles de génération strictes et la vérification post-génération.

**build-pipeline.md :**
- Nouvelle section "Pipeline Vite" (recommandé pour nouveaux projets 2025+) — `vite.config.js` complet avec SCSS natif, ESM, output fixe pour `.libraries.yml`, sourcemaps, minification esbuild
- Comparatif Gulp / Webpack 5 / Vite 5 avec recommandations 2025

**SKILL.md :**
- Description frontmatter étendue : Bootstrap5, SDC, CSS Container Queries, BEM/Storybook, Vite, dark mode, ARIA/WCAG, Symfony Mailer email templates
- Quick Decision Table : entrées `theme-generator`, `Vite (recommandé)`, Gulp/Webpack (legacy), `twig_field_value`

**lessons.md :**
- Leçon `drush cr insuffisant` : suppression de la mention `deleteAll()` sur render cache (risqué en prod) — remplacée par redémarrage container ou Twig debug

---

## v1.1 — 2026-05-14

**Bugs corrigés :**
- `file_create_url()` → `file_url_generator->generateAbsoluteString()` (supprimé D10)
- `{{ is_front_page }}` → `{{ is_front }}` (nom de variable inexistant)
- `:var` placeholder — description corrigée (crée un `<a>`, pas une URL brute)
- `{{ dump(variables) }}` → `{{ dump() }}` (`variables` n'existe pas dans Drupal Twig)
- `views-fields--VIEW-ID.html.twig` → `views-view-fields--VIEW-ID--DISPLAY-ID.html.twig`

**Incohérences corrigées :**
- `libraries-override:` en doublon YAML → regroupé sous une seule clé avec avertissement
- `block--REGION.html.twig` — note ⚠️ ajoutée (non automatique, requiert hook)
- `paragraph--TEXT.html.twig` → `paragraph--text.html.twig` (machine name en minuscules)
- `preprocess: false` commentaire → explication précise (agrégation Drupal, pas SCSS)
- `docker compose exec php drush cache:rebuild && docker compose exec php drush cr` → `docker compose exec php drush cr` (doublon)
- `$entity` dead code dans preprocess_field → utilisé (`getEntityTypeId()`)
- `@classy/css/...` dans stylesheets-remove → exemple neutre sans module supprimé D10
- Underscores vs tirets explication → ajoutée directement dans theme-suggestions.md

**Ajouts :**
- `field.html.twig` template complet dans twig-templates.md
- `{{ drupal_view() }}`, `{{ content.links }}`, `{{ drupal_block() }}` dans twig-templates.md
- `media: print` et `header: true` dans libraries-assets.md
- 4 nouvelles entrées dans Quick Decision Table (SDC, meta tags, CSS suppression, library_info_alter)
- 4 nouvelles leçons dans lessons.md

---

## v1.0 — 2026-05-14

**Création initiale — absorbe drupal-frontend**

### Couverture

**`theme-anatomy.md`**
- Arborescence complète d'un thème
- `.info.yml` complet : régions, libraries, libraries-override, libraries-extend, stylesheets-remove, ckeditor_stylesheets, logo, hidden
- Tableau des base themes (stable9, olivero, claro, false — classy supprimé D10)
- Workflow dev : Twig debug, désactivation agrégation CSS/JS
- Single Directory Components (SDC) — D10.3+ / D11

**`libraries-assets.md`**
- Les 5 niveaux CSS Drupal (base, layout, component, state, theme)
- Options des assets (preprocess, minified, weight, attributes, type: external)
- Dépendances core (`core/once`, `core/drupalSettings`, `core/drupal`)
- 3 méthodes d'attachement : `.info.yml` / Twig / PHP
- `libraries-override` et `libraries-extend` complets
- `drupalSettings` — passage PHP → JS
- Pattern `Drupal.behaviors` moderne avec `once()` (remplace jQuery.once)
- Workflow build tools (Vite, SCSS)

**`twig-templates.md`**
- Tous les filtres Drupal (`|t`, `|render`, `|without`, `|clean_class`, `|clean_id`, `|safe_join`, `|check_markup`, `|date`)
- Fonctions Drupal (`attach_library`, `url`, `path`, `file_url`, `create_attribute`, `dump`)
- L'objet `attributes` — méthodes complètes avec règle absolue
- Accès aux données : `content` (formatter) vs `node` (raw)
- Champs multi-valeurs et entités référencées
- Templates complets : `page.html.twig`, `node.html.twig`, `block.html.twig`
- Héritage Twig (`extends`, `block`, `parent()`)
- `{% trans %}` / `{% plural %}`

**`theme-suggestions.md`**
- Activation Twig Debug + lecture des commentaires HTML
- Patterns complets : page, node, block, field, views, taxonomy, user, paragraph, menu, comment, form, html
- `hook_theme_suggestions_HOOK_alter()` avec exemples réels (page, node, block)
- Priorité des suggestions
- Organisation des dossiers templates

**`preprocess.md`**
- `preprocess_page` : toutes les variables, classes dynamiques, drupalSettings
- `preprocess_node` : date.formatter, is_author, champs référence, attach library
- `preprocess_block` : classes par plugin_id et région
- `preprocess_html` : classes sur `<html>` et `<body>`, rôle utilisateur
- `preprocess_field` : label_display, classes wrapper
- `hook_page_attachments_alter` : meta tags, preconnect
- `hook_library_info_alter` : modifier librairies d'autres modules
- `hook_css_alter` : retirer des CSS au runtime
- Tableau comparatif `build()` vs `preprocess`

**`responsive-images.md`**
- `breakpoints.yml` complet avec `multipliers` pour Retina
- Pipeline complet : Image Styles → Responsive Image Style → Formatter
- WebP natif D10+ dans les Image Styles
- `drupal_image()` dans Twig
- Accès aux propriétés d'image (`alt`, `title`, `width`, `height`, `uri`)
- Lazy loading natif avec `loading="lazy"` et prévention CLS
- Images de fond CSS avec CSS custom properties
- Module `focal_point` — configuration

### Ce que drupal-theming apporte vs drupal-frontend

| Domaine | drupal-frontend | drupal-theming |
|---------|----------------|----------------|
| Drupal Behaviors JS | Non couvert | Pattern complet avec `once()` |
| `core/once` vs jQuery | Non mentionné | Expliqué + raison |
| `drupalSettings` PHP→JS | Non couvert | Complet avec exemple |
| `attributes.addClass()` | Mentionné | Règle absolue + erreurs communes |
| `|check_markup` | Absent | Présent |
| `|clean_id` | Absent | Présent |
| `{% trans %}` / `{% plural %}` | Absent | Présent |
| SDC (Single Dir Components) | Absent | D10.3+/D11 |
| `base theme: classy` D10 | Non averti | Anti-pattern documenté |
| WebP D10+ | Absent | Pipeline complet |
| `focal_point` | Absent | Intégration documentée |
| `hook_css_alter` | Absent | Présent |
| `hook_library_info_alter` | Absent | Présent |
| `preprocess_html` | Absent | Présent |
| Tous les patterns de suggestions | Partiel | Complet (12 types) |

---

## Compatibilité Drupal

| Skill version | Drupal testé | Notes |
|--------------|-------------|-------|
| v1.0 | D10, D11 | `classy` supprimé D10 — `stable9` recommandé |
