---
name: drupal-gin — gin navigation
description: Configurer la nouvelle navigation sidebar Gin (drupal/gin_login) pour Drupal 10.3+ - navigation moderne, personnalisation, et accès programmatique.
---

# Gin Navigation — Sidebar Navigation Moderne

## Contexte — Gin 3.x vs Navigation Core D11

```
Gin 3.x (D10.3+ compatible) :
  → Sidebar navigation intégrée au thème Gin
  → Configurable : icônes + libellés
  → Sous-thème Gin peut surcharger les items

Gin Navigation (module séparé drupal/gin_toolbar) :
  → Toolbar position : horizontal | sidebar | condensed
  → Accessible depuis les Settings Gin

Navigation Module (Core D11+) :
  → Proposition de remplacement de la toolbar Drupal
  → Expérimental D11, susceptible d'évoluer
  → Gin intègre son propre système de navigation
```

---

## Configurer la Navigation Sidebar dans Gin

```
/admin/appearance/settings/gin

→ Gin Toolbar → Layout options :
  ├── Classic (horizontal bar) — barre en haut traditionnelle
  ├── Sidebar — navigation latérale gauche avec icônes + labels
  └── Condensed — sidebar avec icônes uniquement (pour petits écrans)
```

```yaml
# config/install/gin.settings.yml — configurer en code
classic_toolbar: sidebar   # horizontal | sidebar | condensed
secondary_toolbar_enabled: false
sidebar_expand_all: false
```

---

## Personnaliser les Items de Navigation

```php
<?php
// Modifier les éléments de la toolbar/navigation Gin via hook
// (le hook varie selon la version de Gin)

/**
 * Implements hook_toolbar_alter().
 * Modifie les items de la toolbar Drupal utilisée par Gin.
 */
function mon_module_toolbar_alter(array &$items): void {
  // Réorganiser les items
  if (isset($items['administration'])) {
    $items['administration']['#weight'] = -100;
  }

  // Ajouter un item custom
  $items['mon_lien_rapide'] = [
    '#type' => 'toolbar_item',
    '#weight' => 50,
    'tab' => [
      '#type' => 'link',
      '#title' => t('Tableau de bord'),
      '#url' => \Drupal\Core\Url::fromRoute('mon_module.dashboard'),
      '#attributes' => [
        'class' => ['toolbar-icon', 'toolbar-icon-dashboard'],
        'title' => t('Tableau de bord personnalisé'),
      ],
    ],
    '#attached' => [
      'library' => ['mon_module/toolbar-icons'],
    ],
  ];
}
```

---

## Navigation — Icônes dans la Sidebar

```scss
// Dans le sous-thème Gin — CSS pour les icônes de navigation
// mon_admin_theme/css/admin-overrides.css

// Icône custom pour un item de navigation
.toolbar-icon-mon-module::before {
  // Utiliser une icône SVG en background
  background-image: url("../icons/dashboard.svg");
  background-repeat: no-repeat;
  background-size: 16px 16px;
  background-position: center;
  content: '';
  display: inline-block;
  width: 16px;
  height: 16px;
  vertical-align: middle;
  margin-right: 4px;
}
```

---

## Gin Navigation et Layout Builder

```
Gin sidebar navigation + Layout Builder :
  → Pas de conflit — la navigation reste fixe à gauche
  → Le Canvas Layout Builder occupe toute la largeur centrale
  → Pour économiser l'espace : utiliser Gin en mode "condensed"
     lors de l'édition avec Layout Builder

Configuration recommandée pour les sites avec Layout Builder :
  → Gin toolbar : sidebar ou condensed
  → Activer "sticky action buttons" dans Gin settings
  → Utiliser le module "gin_gutenberg" si éditeur Gutenberg est utilisé
```

---

## Gin Toolbar vs Gin Navigation — Différences

| | Gin Toolbar | Navigation Module (D11 core) |
|---|---|---|
| Source | Module contrib `drupal/gin_toolbar` | Core Drupal 11 (expérimental) |
| Compatibilité | D9, D10, D11 | D11 uniquement |
| Personnalisation | Hook `hook_toolbar_alter()` | Hook `hook_menu_local_tasks_alter()` |
| Icônes | CSS background-image | SVG sprites |
| Config UI | Gin settings | Separate config page |

```bash
# Activer gin_toolbar (pour la navigation Gin)
composer require drupal/gin_toolbar
drush en gin_toolbar -y

# Vérifier la compatibilité avec la version de Gin installée
drush pm:list --status=enabled | grep gin
```

---

## Breadcrumbs dans la Navigation Gin

```php
// Personnaliser les breadcrumbs affichés dans la navigation Gin
// Via le service drupal_breadcrumb_builder

// services.yml
services:
  mon_module.breadcrumb_builder:
    class: Drupal\mon_module\BreadcrumbBuilder
    tags:
      - { name: breadcrumb_builder, priority: 100 }
```

---

## Monitoring de l'Accès via Navigation

```bash
# Vérifier quels éléments de navigation sont disponibles
drush php:eval "
\$toolbar = \Drupal::service('toolbar.menu_tree');
// Les éléments de navigation dépendent des permissions de l'utilisateur
\$current_user = \Drupal::currentUser();
echo 'Admin: ' . (\$current_user->hasPermission('administer site configuration') ? 'OUI' : 'NON') . PHP_EOL;
echo 'Access toolbar: ' . (\$current_user->hasPermission('access toolbar') ? 'OUI' : 'NON') . PHP_EOL;
"
```
