---
name: drupal-gin — gin navigation
description: Configurer la navigation sidebar Gin (drupal/gin_toolbar + module Navigation core) pour Drupal 10.3+ - navigation moderne, personnalisation, et accès programmatique.
---

# Gin Navigation — Sidebar Navigation Moderne

## Contexte — 3 systèmes de navigation distincts (ne pas confondre)

```
1. drupal/gin_toolbar (module contrib séparé) :
   → Fournit la toolbar Gin "classique" (style Seven/Claro amélioré)
   → Géré par gin.settings → classic_toolbar: horizontal | vertical | new
   → Requis EN PLUS de drupal/gin (pas un sous-module de gin)

2. Module Navigation (CORE Drupal) :
   → Ajouté en core 10.3 (expérimental), stable en 11.x
   → Sidebar latérale gauche moderne, repliable
   → Module core "navigation" (PAS gin_toolbar, PAS gin_login)
   → Gin s'y intègre : gin.settings → classic_toolbar: new

3. drupal/gin_login (module contrib) :
   → Uniquement la page de connexion brandée
   → AUCUN rapport avec la navigation admin (cf. gin-setup.md)
```

> ⚠️ Erreur fréquente : `gin_login` ne gère PAS la navigation. La sidebar
> moderne D10.3+ vient du module **core `navigation`**, activée côté Gin par
> `classic_toolbar: new`.

---

## Configurer la Navigation Sidebar dans Gin

```
/admin/appearance/settings/gin

→ Gin Toolbar → Navigation :
  ├── Horizontal — barre en haut traditionnelle (style Claro)
  ├── Vertical — toolbar latérale gauche fournie par gin_toolbar
  └── New (Navigation module) — sidebar moderne du module core navigation (D10.3+)
```

```yaml
# config/sync/gin.settings.yml — configurer en code
# Valeurs réelles de classic_toolbar : horizontal | vertical | new
# 'new' = délègue au module core "navigation" (doit être activé : drush en navigation -y)
settings:
  classic_toolbar: new
  secondary_toolbar_frontend: false
  show_user_theme_settings: true
```

> Vérifier les valeurs exactes pour la version installée :
> `drush config:get gin.settings settings.classic_toolbar`

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
Sidebar navigation + Layout Builder :
  → Pas de conflit — la navigation reste fixe à gauche, repliable
  → Le canvas Layout Builder occupe toute la largeur centrale
  → Replier la sidebar (module navigation) pour gagner de l'espace en édition

Configuration recommandée pour les sites avec Layout Builder :
  → classic_toolbar: new (sidebar repliable) ou vertical
  → Activer "sticky action buttons" dans Gin settings
  → Utiliser le module "gin_gutenberg" si l'éditeur Gutenberg est utilisé
```

---

## Gin Toolbar vs Gin Navigation — Différences

| | Gin Toolbar | Module Navigation (core) |
|---|---|---|
| Source | Module contrib `drupal/gin_toolbar` | Core Drupal (10.3 expérimental, 11.x stable) |
| Compatibilité | D9, D10, D11 | D10.3+ / D11 |
| Personnalisation | Hook `hook_toolbar_alter()` | Bloc « Navigation », liens de menu admin + `hook_block_build_alter` |
| Icônes | CSS background-image | Icônes du système d'icônes core (`*.icons.yml`) |
| Config UI | Gin settings | `/admin/config/user-interface/navigation-blocks` |

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
# Les éléments de navigation dépendent des permissions de l'utilisateur courant.
# 'access toolbar' = module toolbar / gin_toolbar ; 'access navigation' = module core navigation.
drush php:eval "
\$u = \Drupal::currentUser();
echo 'admin config:   ' . (\$u->hasPermission('administer site configuration') ? 'OUI' : 'NON') . PHP_EOL;
echo 'access toolbar: ' . (\$u->hasPermission('access toolbar') ? 'OUI' : 'NON') . PHP_EOL;
echo 'access navigation: ' . (\$u->hasPermission('access navigation') ? 'OUI' : 'NON') . PHP_EOL;
"
```

> Masquer un item de toolbar selon le rôle/permission : faire la vérification
> `$account->hasPermission(...)` **dans** `hook_toolbar_alter()` avant d'ajouter
> l'item, et ajouter le cache context `'user.permissions'` via `#cache`.
