---
name: drupal-views — cache
description: Configuration et optimisation du cache des Views Drupal - tag-based caching, time-based, cache tags custom, invalidation, et impact des relations sur le cache.
---

# Views Cache — Configuration & Optimisation

## Types de Cache Views

```
Views UI → Advanced → Caching → Change

├── None
│     Pas de mise en cache — requête SQL à chaque chargement
│     Uniquement pour les Views ultra-dynamiques (temps réel)
│
├── Time-based caching
│     Cache query results : N secondes
│     Cache rendered output : N secondes
│     Invalidation : uniquement par expiration du TTL
│
└── Tag-based caching (recommandé)
      Cache invalide automatiquement quand les entités associées changent
      Pas de TTL — le cache est valide tant que les données n'ont pas changé
```

**Recommandation :** Tag-based pour toutes les Views qui affichent des entités. Time-based uniquement si les données changent rarement et que l'invalidation granulaire n'est pas critique.

---

## Tag-Based Caching — Fonctionnement

Drupal associe automatiquement des cache tags aux Views selon leur configuration :

```
View de nœuds → tags : ['node_list', 'node:42', 'node:43', ...]
View d'utilisateurs → tags : ['user_list', 'user:1', ...]
View de taxonomy → tags : ['taxonomy_term_list', 'taxonomy_term:5', ...]
```

**Quand un nœud est modifié (`$node->save()`) :**
1. Drupal invalide automatiquement `['node:42', 'node_list']`
2. Toutes les caches qui ont ces tags sont invalidées
3. La prochaine requête régénère la Vue

---

## Configurer le Cache d'une View via YAML

```yaml
# config/sync/views.view.articles_recents.yml
display:
  default:
    cache_metadata:
      max-age: -1       # -1 = PERMANENT (invalidation par tags)
      contexts:
        - 'languages:language_interface'
        - 'url.query_args'
      tags:
        - node_list
```

---

## Ajouter des Cache Tags Custom

```php
/**
 * Implements hook_views_pre_render().
 *
 * Ajouter des tags de cache supplémentaires à une View.
 * Utiliser quand la View dépend de données non détectées automatiquement.
 */
function mon_module_views_pre_render(\Drupal\views\ViewExecutable $view): void {
  if ($view->id() !== 'commandes_dashboard') {
    return;
  }

  // Tags d'entités custom (table non-Drupal)
  $view->element['#cache']['tags'][] = 'mon_module_commande_list';

  // Invalider aussi quand la config change
  $view->element['#cache']['tags'][] = 'config:mon_module.settings';

  // Tags calculés selon les résultats
  foreach ($view->result as $row) {
    if (isset($row->mon_module_commandes_id)) {
      $view->element['#cache']['tags'][] = 'mon_module_commande:' . $row->mon_module_commandes_id;
    }
  }

  // Contexts — varier le cache selon ces dimensions
  $view->element['#cache']['contexts'][] = 'user.roles';

  // Max-age en secondes (en plus des tags)
  $view->element['#cache']['max-age'] = 300;  // 5 minutes max même si tags OK
}
```

---

## Invalider le Cache d'une View Manuellement

```php
// Invalider toutes les caches de la View (tag Views)
\Drupal\Core\Cache\Cache::invalidateTags(['config:views.view.ma_view']);

// Invalider via les données qu'elle affiche
\Drupal\Core\Cache\Cache::invalidateTags(['node_list']);
\Drupal\Core\Cache\Cache::invalidateTags(['mon_module_commande_list']);

// Depuis Drush
drush php:eval "\Drupal\Core\Cache\Cache::invalidateTags(['node_list']);"
drush php:eval "\Drupal\Core\Cache\Cache::invalidateTags(['config:views.view.articles_recents']);"

// Vider uniquement le render cache (pas la query cache)
drush php:eval "\Drupal::cache('render')->invalidateTags(['config:views.view.articles_recents']);"
```

---

## Cache et Vues avec Filtres Exposés

Les filtres exposés créent des variantes de cache selon les query parameters.

```php
// Cache context pour les filtres exposés dans l'URL
$view->element['#cache']['contexts'][] = 'url.query_args';

// Cache context pour un filtre exposé spécifique
$view->element['#cache']['contexts'][] = 'url.query_args:type';
$view->element['#cache']['contexts'][] = 'url.query_args:langcode';
```

**Impact :** chaque combinaison de filtres = une entrée de cache séparée. Sur des Views avec de nombreux filtres exposés, le nombre de variantes peut exploser → penser à limiter les combinaisons ou augmenter le TTL.

---

## Cache des Vues en Block

```php
// BlockBase::getCacheTags() — pour un bloc qui contient une View
public function getCacheTags(): array {
  $view = Views::getView('articles_recents');
  if (!$view) {
    return parent::getCacheTags();
  }

  $view->setDisplay('block_1');
  $view->initDisplay();

  // Récupérer les tags de la View
  $view_tags = $view->getCacheMetadata()->getCacheTags();

  return \Drupal\Core\Cache\Cache::mergeTags(
    parent::getCacheTags(),
    $view_tags
  );
}
```

---

## Performance — Analyser les Requêtes Views

```php
// Activer le logging des queries Views (dev uniquement)
function mon_module_views_query_alter(\Drupal\views\ViewExecutable $view, $query): void {
  // Logger le temps d'exécution
  $start = microtime(TRUE);

  // La query s'exécute — accéder au temps après post_execute
}

function mon_module_views_post_execute(\Drupal\views\ViewExecutable $view): void {
  \Drupal::logger('mon_module')->debug(
    'View @view (@display): @count résultats',
    [
      '@view' => $view->id(),
      '@display' => $view->current_display,
      '@count' => count($view->result),
    ]
  );
}
```

---

## Checklist Cache Views

```
[ ] Tag-based caching activé (pas None, pas time-based si entités)
[ ] Cache contexts configurés (user.roles si contenu personnalisé)
[ ] hook_views_pre_render() pour les tags custom si données non-entités
[ ] Varnish configuré pour invalider sur X-Drupal-Cache-Tags
[ ] AJAX Views activé si filtres exposés (évite les problèmes de cache partiel)
[ ] Views de dashboard/stats → time-based avec TTL adapté
[ ] Views anonymes → Page Cache actif (tags Views automatiquement inclus)
```
