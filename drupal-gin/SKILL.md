---
name: drupal-gin
description: Use when configuring the Gin admin theme for Drupal - installing and setting Gin as the default admin theme, creating a Gin sub-theme for custom branding (CSS custom variables --gin-color-primary for colors, logo, custom CSS overrides), configuring Gin settings (toolbar position horizontal/sidebar/condensed, navigation mode, accent color, dark mode), using hook_toolbar_alter() to add custom navigation items with SVG icons, customizing the content editing form layout with hook_form_NODE_TYPE_edit_form_alter() to move fields to the sidebar meta panel, installing gin_login for branded login page, configuring the Gin navigation module for the new sidebar navigation in D10.3+, or migrating from Adminimal/Seven/Claro to Gin in Drupal 9-11+
---

# Drupal Gin Admin Theme — Référence Complète

## Overview

Référentiel complet du thème d'administration Gin (drupal/gin) pour Drupal 9-11+ : installation, configuration, sous-thème personnalisé, couleurs/logo custom, mise en page des formulaires d'édition, modules complémentaires. Gin est devenu le thème admin de référence sur les projets Drupal modernes.

## 🎯 Pourquoi Gin ?

```
Seven (ancien thème admin core) :
  ❌ UI datée, peu d'ergonomie
  ❌ Plus maintenu activement
  ❌ Pas de mode sombre, pas de sidebar navigation

Claro (thème admin core actuel) :
  ✅ Moderne, accessible
  ⚠️ Limité en personnalisation
  ⚠️ Pas de sidebar navigation

Gin (contrib) :
  ✅ Toolbar horizontale OU sidebar navigation
  ✅ Mode sombre
  ✅ Couleur d'accent configurable par instance
  ✅ Mise en page éditoriale optimisée (méta en sidebar)
  ✅ Gin Login (page de connexion branded)
  ✅ Actif D11 day-1 support
```

---

## Quick Decision Table

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Installer Gin | `composer require drupal/gin drupal/gin_toolbar` | [gin-setup.md](gin-setup.md) |
| Configurer la nouvelle navigation sidebar (D10.3+) | Gin settings → Toolbar → Sidebar | [gin-navigation.md](gin-navigation.md) |
| Personnaliser les items de navigation Gin | `hook_toolbar_alter()` | [gin-navigation.md](gin-navigation.md) |
| Icône custom dans la sidebar navigation | CSS background-image sur `.toolbar-icon-NOM` | [gin-navigation.md](gin-navigation.md) |
| Définir Gin comme thème admin | `drush config:set system.theme admin gin` | [gin-setup.md](gin-setup.md) |
| Activer la navigation sidebar | Gin settings → Toolbar → Classic Gin Navigation | [gin-setup.md](gin-setup.md) |
| Activer le mode sombre | Gin settings → Appearance → Dark mode | [gin-setup.md](gin-setup.md) |
| Choisir la couleur d'accent | Gin settings → Accent color (preset ou custom) | [gin-setup.md](gin-setup.md) |
| Logo custom dans l'admin | Gin settings → Logo → Custom logo | [gin-setup.md](gin-setup.md) |
| Gin Login (page de connexion branded) | `composer require drupal/gin_login` | [gin-setup.md](gin-setup.md) |
| Créer un sous-thème Gin | `.info.yml` avec `base theme: gin` | [gin-subtheme.md](gin-subtheme.md) |
| CSS custom dans l'administration | Sous-thème Gin + custom CSS | [gin-subtheme.md](gin-subtheme.md) |
| Formulaire d'édition : méta en sidebar | Gin → Content form → Meta sidebar | [gin-subtheme.md](gin-subtheme.md) |
| Grouper les champs en onglets | Module `field_group` compatible Gin | [gin-subtheme.md](gin-subtheme.md) |
| Gin Toolbar (navigation améliorée) | `composer require drupal/gin_toolbar` | [gin-setup.md](gin-setup.md) |
| Layout Builder dans Gin | Compatible natif | [gin-subtheme.md](gin-subtheme.md) |
| Exporter la config Gin | `drush cex` → `gin.settings.yml` | [gin-setup.md](gin-setup.md) |
| **Override couleur primaire Gin (CSS vars)** | `--gin-color-primary: #1a5276` dans sous-thème CSS | [gin-subtheme.md](gin-subtheme.md) |
| **Ajouter un item dans la toolbar Drupal/Gin** | `hook_toolbar_alter()` dans .module | [gin-navigation.md](gin-navigation.md) |
| **Icône custom SVG dans la sidebar navigation** | CSS `.toolbar-icon-MON-MODULE::before { background-image: url(...) }` | [gin-navigation.md](gin-navigation.md) |
| **Déplacer des champs en sidebar via PHP** | `hook_form_node_TYPE_edit_form_alter()` + `$form['field']['#group'] = 'meta'` | [gin-subtheme.md](gin-subtheme.md) |
| **Gin Login — brander la page de connexion** | `composer require drupal/gin_login` + config logo | [gin-setup.md](gin-setup.md) |
| Gin sidebar vs condensed vs horizontal | Gin settings → Toolbar → Layout options | [gin-navigation.md](gin-navigation.md) |
| Breadcrumbs custom dans la navigation Gin | `BreadcrumbBuilder` service avec priority 100 | [gin-navigation.md](gin-navigation.md) |
| **Référence complète CSS Variables Gin** | `--gin-color-primary`, `--gin-color-button`, `--gin-bg-header`, `--gin-font-family` | [gin-subtheme.md](gin-subtheme.md) |
| **Forcer le mode sombre par défaut** | `gin.settings.yml` → `preset: dark` + `classic_toolbar: false` | [gin-setup.md](gin-setup.md) |
| **Gin + Content Moderation (badge d'état dans le formulaire)** | Compatible natif — les états apparaissent dans le panneau meta Gin | [gin-subtheme.md](gin-subtheme.md) |
| **Gin + Paragraphs — afficher les champs para dans la sidebar** | `hook_form_node_TYPE_edit_form_alter()` → `$form['field_paragraphs']['#group'] = 'meta'` | [gin-subtheme.md](gin-subtheme.md) |
| **Gin + field_group — onglets / accordéons dans les formulaires** | Module `field_group` ≥ 3.4 compatible Gin — layout `tabs`, `accordion`, `fieldset` | [gin-subtheme.md](gin-subtheme.md) |
| **Gin + CKEditor 5 — toolbar placement optimisé** | Gin positionne CKEditor 5 en pleine largeur natif — aucune config spéciale | [gin-setup.md](gin-setup.md) |
| **Gin + Views admin — exposed filters au style Gin** | Module `gin_exposed_form_block` — les filtres exposés apparaissent en sidebar Gin | [gin-navigation.md](gin-navigation.md) |
| **Désactiver les animations Gin (accessibilité)** | `@media (prefers-reduced-motion: reduce)` dans le sous-thème CSS | [gin-subtheme.md](gin-subtheme.md) |
| **Override favicon dans l'administration** | `gin.settings.yml` → `favicon.use_default: false` + chemin custom | [gin-setup.md](gin-setup.md) |
| **Masquer des items de navigation pour certains rôles** | `hook_toolbar_alter()` + vérification `$current_user->hasPermission()` | [gin-navigation.md](gin-navigation.md) |
| **Migrer de Claro → Gin** | `drush config:set system.theme admin gin` + exporter gin.settings.yml + créer sous-thème | [gin-setup.md](gin-setup.md) |
| **Migrer d'Adminimal → Gin** | Désinstaller adminimal, installer gin + gin_toolbar, vérifier les surcharges CSS custom | [gin-setup.md](gin-setup.md) |
| **Gin + sélecteur de langue dans la navigation** | Module `language` → bloc `language_block` positionné dans toolbar Gin | [gin-navigation.md](gin-navigation.md) |
| **Personnaliser la couleur du header admin par environnement** | `$config['gin.settings']['preset']` dans `settings.local.php` → rouge en prod, bleu en staging | [gin-setup.md](gin-setup.md) |

## Migration vers Gin

### Depuis Claro (recommandé D10/D11)
```bash
composer require drupal/gin drupal/gin_toolbar drupal/gin_login
drush config:set system.theme admin gin
drush config:set system.theme login gin_login  # optionnel
drush cr
drush cex  # committer gin.settings.yml
```

### Depuis Seven / Adminimal (D9+)
1. Désinstaller l'ancien thème : `drush theme:uninstall seven`
2. Supprimer les surcharges CSS dans `settings.php` (`ADMINIMAL_*` variables)
3. Installer Gin + créer un sous-thème pour les customisations
4. Inspecter `hook_form_alter` qui ciblaient `.adminimal-*` — renommer en `.gin-*`
5. Vérifier que `field_group` est ≥ 3.4 (compatibilité Gin requise)

### Checklist post-migration
- [ ] Exporter `gin.settings.yml` et le committer
- [ ] Créer un sous-thème Gin pour les customisations CSS (ne jamais modifier Gin directement)
- [ ] Tester le formulaire d'édition des content types principaux
- [ ] Vérifier que les views admin s'affichent correctement
- [ ] Tester le Layout Builder si activé

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| Modifier le thème Gin directement | Créer un sous-thème Gin | Modifications perdues à la mise à jour |
| CSS admin dans le thème frontend | CSS dédié dans le sous-thème Gin | Les styles admin affectent le frontend |
| Gin non exporté en config | `drush cex` → committer `gin.settings.yml` | Config différente entre environnements |
| Gin + Adminimal simultanément | Un seul thème admin actif | Conflits CSS |
| Garder Claro sur D10/D11 | Migrer vers Gin (sidebar nav, ergonomie éditoriale, dark mode) | Gin est devenu le standard agence |
| Charger les assets frontend dans le sous-thème Gin | Librairies admin uniquement dans le sous-thème Gin, assets frontend dans le thème front | Assets frontend en double sur toutes les pages admin |
| `gin.settings.yml` géré uniquement en DB | Toujours exporter et committer `gin.settings.yml` | Config différente en prod → expérience admin inconsistante |

## Évolution par Version Majeure

| Feature | D9 | D10 | D11 |
|---------|----|----|-----|
| Gin module | ✅ | ✅ | ✅ |
| Gin Navigation (sidebar) | contrib | ✅ core-like | ✅ |
| Gin Login | contrib | contrib | contrib |
| Mode sombre | ✅ | ✅ | ✅ |
| D11 support day-1 | ✅ | ✅ | ✅ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Problèmes Gin découverts en projet.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions.

## See Also

- `drupal-theming` — Base themes, .info.yml, CSS/JS libraries, BEM, build pipeline
- `drupal-config` — Export/import config Gin (gin.settings.yml)
- `drupal-content-modeling` — Mise en page des formulaires d'édition, field_group, Layout Builder
- `drupal-core` — `hook_toolbar_alter()`, `hook_form_alter()`, BreadcrumbBuilder
- `drupal-deployment` — Environnements différents, couleur header par environnement (`settings.local.php`)
