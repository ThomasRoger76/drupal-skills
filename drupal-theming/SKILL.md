---
name: drupal-theming
description: Use when creating or customizing Drupal themes, writing Twig templates, implementing preprocess hooks in .theme files, declaring CSS/JS libraries (.libraries.yml), configuring responsive images with breakpoints.yml, debugging template suggestions via Twig debug, overriding base themes, building with Bootstrap5 sub-theme, compiling SCSS with Gulp/Webpack/Vite, implementing BEM methodology with Storybook component documentation, using CSS Container Queries or Cascade Layers, creating Single Directory Components (SDC), implementing dark mode with CSS variables, building accessible themes (ARIA, WCAG), or templating emails with Symfony Mailer in Drupal 8-11+
---

# Drupal Theming — Architecture & Référence Complète

## Overview

Référentiel complet du theming Drupal 8-11+ : anatomie d'un thème, gestion des assets, moteur Twig, suggestions de templates, preprocess PHP, images responsives. Absorbe `drupal-frontend`.

> **Pattern dominant agences françaises :** Bootstrap 5 via `drupal/bootstrap5` avec sous-thème custom (80%+ des projets). Le fichier [bootstrap5.md](bootstrap5.md) couvre la création complète d'un sous-thème Bootstrap 5 avec SCSS, templates Twig Bootstrap, menus responsive, carrousels Paragraphs, et formulaires avec classes BS5.

## Quick Decision Table

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Créer un thème from scratch | Structure + `.info.yml` | [theme-anatomy.md](theme-anatomy.md) |
| Définir des régions | `regions:` dans `.info.yml` | [theme-anatomy.md](theme-anatomy.md) |
| Choisir / hériter d'un base theme | `base theme:` — stable9 / olivero / false | [theme-anatomy.md](theme-anatomy.md) |
| Charger CSS/JS sur toutes les pages | `libraries:` dans `.info.yml` | [libraries-assets.md](libraries-assets.md) |
| Charger CSS/JS conditionnellement en Twig | `{{ attach_library('mon_theme/lib') }}` | [libraries-assets.md](libraries-assets.md) |
| Charger CSS/JS depuis PHP (preprocess) | `$variables['#attached']['library'][]` | [libraries-assets.md](libraries-assets.md) |
| Remplacer une librairie core/contrib | `libraries-override:` dans `.info.yml` | [libraries-assets.md](libraries-assets.md) |
| Étendre une librairie existante | `libraries-extend:` dans `.info.yml` | [libraries-assets.md](libraries-assets.md) |
| Passer des données PHP → JavaScript | `drupalSettings` | [libraries-assets.md](libraries-assets.md) |
| JS sans jQuery (D9+) | `core/once` + vanilla JS | [libraries-assets.md](libraries-assets.md) |
| Savoir quel template modifier | Twig Debug — commentaires HTML | [theme-suggestions.md](theme-suggestions.md) |
| Surcharger le template d'un nœud | `node--TYPE--VIEW-MODE.html.twig` | [theme-suggestions.md](theme-suggestions.md) |
| Ajouter une suggestion conditionnelle | `hook_theme_suggestions_HOOK_alter()` | [theme-suggestions.md](theme-suggestions.md) |
| Accéder proprement aux valeurs de champs en Twig | Module `twig_field_value` — `field.value`, `field.0.value` | [twig-templates.md](twig-templates.md) |
| Exclure un champ du rendu | `{{ content\|without('field_image') }}` | [twig-templates.md](twig-templates.md) |
| Ajouter une classe CSS dynamique | `{{ attributes.addClass('ma-classe') }}` | [twig-templates.md](twig-templates.md) |
| Filtres Drupal (|t, |render, |clean_class…) | Référence filtres Twig | [twig-templates.md](twig-templates.md) |
| Préparer des variables avant Twig | `hook_preprocess_HOOK` dans `.theme` | [preprocess.md](preprocess.md) |
| Classes CSS dynamiques selon contexte | Preprocess + `$variables['attributes']` | [preprocess.md](preprocess.md) |
| Images responsive multi-breakpoint | `breakpoints.yml` + Responsive Image Styles | [responsive-images.md](responsive-images.md) |
| WebP automatique | Image Styles D10+ | [responsive-images.md](responsive-images.md) |
| Supprimer CSS core/contrib | `stylesheets-remove:` / `hook_css_alter` | [libraries-assets.md](libraries-assets.md) |
| Modifier librairies d'autres modules | `hook_library_info_alter` | [preprocess.md](preprocess.md) |
| Ajouter meta tags / preconnect | `hook_page_attachments_alter` | [preprocess.md](preprocess.md) |
| Créer un composant réutilisable (D10.3+) | Single Directory Components | [theme-anatomy.md](theme-anatomy.md) |
| Utiliser Bootstrap 5 comme base | `drupal/bootstrap5` + sous-thème custom | [bootstrap5.md](bootstrap5.md) |
| Créer un sous-thème Bootstrap 5 | `.info.yml` avec `base theme: bootstrap5` | [bootstrap5.md](bootstrap5.md) |
| Surcharger des templates Bootstrap5 | Copier dans `templates/` du sous-thème | [bootstrap5.md](bootstrap5.md) |
| Générer un thème Drupal 11 complet from scratch | Agent `/drupal-generate-theme` | [agents/theme-generator.md](agents/theme-generator.md) |
| Compiler SCSS → CSS (nouveaux projets) | **Vite** (recommandé 2025+) | [build-pipeline.md](build-pipeline.md) |
| Compiler SCSS → CSS (projets legacy) | Gulp ou Webpack | [build-pipeline.md](build-pipeline.md) |
| Service Docker Node.js pour le thème | `node:22-alpine` + `npm run watch` | [build-pipeline.md](build-pipeline.md) |
| Twig 3 arrow functions (\|map, \|filter, \|reduce) | Arrow functions `u => u.name` | [twig3-accessibility.md](twig3-accessibility.md) |
| Navigation accessible avec ARIA | `aria-current`, `aria-expanded` | [twig3-accessibility.md](twig3-accessibility.md) |
| Images avec alt WCAG | `alt=""` décoratif, alt descriptif | [twig3-accessibility.md](twig3-accessibility.md) |
| Template email HTML (Symfony Mailer) | CSS inline, structure HTML email | [email-templates.md](email-templates.md) |
| apply block Twig 3 | `{% apply upper %}{% endapply %}` | [twig3-accessibility.md](twig3-accessibility.md) |
| Nommer les classes CSS selon BEM | `.block__element--modifier` dans Twig | [bem-storybook.md](bem-storybook.md) |
| CSS responsive selon le conteneur (pas la viewport) | CSS Container Queries | [css-moderne.md](css-moderne.md) |
| Dark mode auto (prefers-color-scheme) | CSS Variables + @media dark | [css-moderne.md](css-moderne.md) |
| CSS adapté RTL/LTR automatiquement | CSS Logical Properties | [css-moderne.md](css-moderne.md) |
| Organiser CSS avec priorités explicites | CSS Cascade Layers @layer | [css-moderne.md](css-moderne.md) |
| Documenter et tester les composants visuellement | Storybook + HTML stories | [bem-storybook.md](bem-storybook.md) |
| Service Docker pour Storybook | `node:22-alpine` port 6006 | [bem-storybook.md](bem-storybook.md) |
| État CSS via BEM modifier | `.card--active` vs `.is-active` | [bem-storybook.md](bem-storybook.md) |
| Accessibilité WCAG 2.1 AA (ARIA, keyboard, contrast) | alt text obligatoire, focus visible, landmarks | [accessibility.md](accessibility.md) |
| Screen reader — contenu visible uniquement | `.visually-hidden` + `aria-live` pour les annonces | [accessibility.md](accessibility.md) |
| Tester l'accessibilité automatiquement | axe-core, Lighthouse, editoria11y module | [accessibility.md](accessibility.md) |
| Skip link — navigation clavier rapide | Premier élément de page = `<a href="#main-content">` | [accessibility.md](accessibility.md) |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Raison |
|---------------------|------------------|--------|
| `class="{{ attributes.class }} ma-classe"` | `{{ attributes.addClass('ma-classe') }}` | Écrase les classes Drupal système |
| `core/jquery` en dépendance par défaut | `core/once` + vanilla JS | jQuery n'est plus chargé par défaut D9+ |
| `jQuery(document).ready(function() {})` | `(function(Drupal, once) { Drupal.behaviors.X = {...} })(Drupal, once)` | Pattern Drupal behaviors obligatoire |
| Logique complexe dans les templates Twig | Déplacer dans `preprocess_HOOK` | Twig doit rester déclaratif |
| `base theme: classy` en D10+ | `base theme: stable9` ou `base theme: false` | `classy` supprimé en D10 |
| `{{ node.field_image.value }}` | `{{ content.field_image }}` ou `{{ node.field_image.entity.uri.value }}` | `content` applique le formatter, `node` donne le raw |
| `!important` dans CSS | Comprendre la cascade, utiliser les niveaux CSS | Indique une mauvaise architecture CSS |
| Charger toutes les librairies globalement | Attacher conditionnellement | Perfs — Drupal charge à la demande |
| `$variables['theme_hook_suggestions'][]` | `hook_theme_suggestions_HOOK_alter()` | Ancienne API, toujours fonctionnelle mais déconseillée |

## Évolution par Version Majeure

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| jQuery chargé par défaut | ✅ | ⚠️ opt-in | ❌ | ❌ |
| Base theme `classy` | ✅ | ✅ | ❌ supprimé | ❌ |
| Base theme `stable9` | ❌ | ✅ | ✅ recommandé | ✅ |
| Base theme `olivero` (front) | ❌ | ❌ | ✅ stable | ✅ |
| Base theme `claro` (admin) | ❌ | ✅ stable | ✅ | ✅ |
| `core/once` (remplace jQuery once) | ❌ | ✅ | ✅ standard | ✅ |
| WebP natif (Image Styles) | ❌ | ❌ | ✅ | ✅ |
| Single Directory Components (SDC) | ❌ | ❌ | ✅ expérimental | ✅ stable |
| `{% trans %}` / `{% plural %}` Twig | ✅ | ✅ | ✅ | ✅ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Bugs trouvés en usage réel. Ajouter une entrée après chaque correction.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions (v1.0 courante).

## See Also

- `drupal-core` — hooks, services, rendering system côté module, TrustedCallbackInterface pour les callbacks `#pre_render`
- `drupal-config` — Config Management (config/install du thème)
- `drupal-security` — échappement XSS dans Twig, `#markup` vs `#plain_text`
- `drupal-deployment` — `drush cr`, `drush twig:debug`, cache rebuild
- `drupal-docker` — Service Node.js Docker Compose pour le build pipeline
- `drupal-performance` — agrégation CSS/JS, WebP, Core Web Vitals, lazy loading images
- `drupal-content-modeling` — templates Paragraphs, Layout Builder, nodes, Block Content types
- `drupal-sdc` — Single Directory Components, Storybook, props et slots
- `drupal-gin` — Admin theme Gin pour personnaliser l'interface d'administration
