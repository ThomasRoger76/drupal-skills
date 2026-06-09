---
name: drupal-performance — cache system
description: Architecture complète du système de cache Drupal (Page Cache, Dynamic Page Cache, Render Cache, BigPipe), configuration, backends, désactivation en développement, et debug.
---

# Système de Cache Drupal — Référence Complète

## Architecture des Couches de Cache

```
Requête HTTP
     ↓
[Varnish/CDN]  ← Avant PHP, cache HTTP complet (reverse proxy)
     ↓ MISS
[Page Cache]   ← Réponse HTTP complète pour utilisateurs anonymes (module core)
     ↓ connecté
[Dynamic Page Cache] ← Parties variables mises en cache séparément (module core)
     ↓
[BigPipe]      ← Placeholders streamés en SSE après le HTML initial
     ↓
[Render Cache] ← Composants (blocs, entités, render arrays)
     ↓
[Entity Cache] ← Données d'entités brutes (via entity_cache ou natif D10)
     ↓
[Database]     ← Source de données
```

---

## Page Cache — Utilisateurs Anonymes

Le Page Cache stocke la réponse HTTP **complète** (HTML + headers) pour les utilisateurs non connectés. Activé par défaut, aucune configuration requise.

```php
// Vérifier si Page Cache est actif
\Drupal::moduleHandler()->moduleExists('page_cache'); // TRUE si actif

// Forcer l'exclusion d'une page du Page Cache — pattern Drupal natif.
// Dans un controller : retourner une réponse avec max-age 0 via CacheableMetadata.
// C'est ce que lit le module page_cache (pas seulement les headers HTTP bruts).
use Drupal\Core\Cache\CacheableMetadata;
$build['#cache']['max-age'] = 0;   // sur le render array → page jamais mise en Page Cache

// Sur une CacheableResponse (controller renvoyant un objet Response) :
$metadata = new CacheableMetadata();
$metadata->setCacheMaxAge(0);
$response->addCacheableDependency($metadata);

// Sur une réponse Symfony brute (non cacheable) :
$response->headers->set('Cache-Control', 'no-cache, no-store, must-revalidate');
$response->setMaxAge(0);
$response->setSharedMaxAge(0);
```

**Quand une page n'est PAS mise en Page Cache :**
- Utilisateur connecté
- Cookie de session présent
- Réponse avec code != 200
- `Cache-Control: no-cache` dans la réponse

**Invalidation :** par cache tags. Quand `node:42` est modifié, toutes les pages qui avaient ce tag sont invalidées.

---

## Dynamic Page Cache — Utilisateurs Connectés

Dynamic Page Cache met en cache les **parties invariantes** d'une page et les combine avec les parties variables (blocs personnalisés) au moment du rendu.

```php
// Déclarer qu'un render array est spécifique à l'utilisateur
$build['mon_bloc'] = [
  '#markup' => $this->t('Bonjour @name', ['@name' => $user->getDisplayName()]),
  '#cache' => [
    // Le DPC met BIEN en cache ce bloc — une variante distincte par utilisateur.
    // 'user' est très granulaire : préférer 'user.roles' si le contenu ne dépend
    // que du rôle. C'est 'max-age' => 0 (et non le context 'user') qui exclut du DPC.
    'contexts' => ['user'],
    'tags' => ['user:' . $user->id()],
    'max-age' => 3600,
  ],
];

// Déclarer un render array comme variant par URL
$build['breadcrumb'] = [
  // ...
  '#cache' => [
    'contexts' => ['url'],      // Différent par URL
    'max-age' => Cache::PERMANENT,
  ],
];
```

---

## BigPipe — Streaming des Placeholders

BigPipe permet d'envoyer le HTML principal immédiatement et de charger les blocs personnalisés en SSE (Server-Sent Events) après. L'utilisateur voit la page quasi-instantanément même si certains blocs sont lents.

```php
// Un render array avec un #lazy_builder est automatiquement géré par BigPipe
$build['mon_bloc_lent'] = [
  '#lazy_builder' => [
    'mon_module.lazy_builder:buildBlocLent',  // Service::méthode
    [$user_id],                               // Arguments (séquence scalaire)
  ],
  '#create_placeholder' => TRUE,             // Forcer BigPipe pour ce composant
  '#cache' => [
    'contexts' => ['user'],
    'max-age' => 0,                          // Toujours recalculé (perso)
  ],
];

// Service lazy builder
// src/LazyBuilder/MonLazyBuilder.php
class MonLazyBuilder implements TrustedCallbackInterface {
  public function buildBlocLent(int $user_id): array {
    // Logique lente (appel API externe, requête complexe...)
    return [
      '#markup' => $this->t('Données personnalisées pour @uid', ['@uid' => $user_id]),
    ];
  }

  public static function trustedCallbacks(): array {
    return ['buildBlocLent'];
  }
}
```

**BigPipe est désactivé si :**
- Le client ne supporte pas SSE (rare)
- `$config['big_pipe.settings']['enabled'] = FALSE`
- La réponse est un JSON/REST endpoint

---

## Render Cache — Composants Individuels

```php
// Déclarer le cache sur n'importe quel render array
$build = [
  '#theme' => 'node',
  '#node' => $node,
  '#view_mode' => 'teaser',
  '#cache' => [
    // Cache tags — invalider quand ces entités changent
    'tags' => $node->getCacheTags(),           // ['node:42']

    // Cache contexts — varier selon ces dimensions
    'contexts' => [
      'languages',                             // Différent par langue
      'url.query_args:page',                   // Différent par numéro de page
      'user.roles',                            // Différent par rôles (groupes)
      // 'user' = différent par utilisateur (très granulaire, à éviter sur blocs)
    ],

    // Max-age — TTL en secondes
    'max-age' => 3600,                         // 1 heure
    // Cache::PERMANENT = pas d'expiration (invalidation par tags uniquement)
    // 0 = jamais mis en cache
  ],
];
```

---

## Cache Backends — Bins Drupal

```php
// Les bins de cache Drupal
\Drupal::cache()                    // cache_default — usage général
\Drupal::cache('render')            // cache_render — render arrays
\Drupal::cache('data')              // cache_data — données temporaires
\Drupal::cache('bootstrap')         // cache_bootstrap — bootstrap Drupal
\Drupal::cache('config')            // cache_config — configuration active
\Drupal::cache('discovery')         // cache_discovery — plugin discovery
\Drupal::cache('page')              // cache_page — Page Cache
\Drupal::cache('dynamic_page_cache') // Dynamic Page Cache

// Lire depuis un cache bin
$cid = 'mon_module:donnees:' . $id;
$cached = \Drupal::cache('data')->get($cid);
if ($cached !== FALSE) {
  return $cached->data;
}

// Écrire dans le cache
$data = $this->computeExpensiveData($id);
\Drupal::cache('data')->set(
  $cid,
  $data,
  Cache::PERMANENT,         // ou timestamp Unix d'expiration
  ['node:' . $node->id()]   // cache tags
);

// Invalider par tags (toutes les caches qui ont ce tag)
\Drupal\Core\Cache\Cache::invalidateTags(['node:42', 'node_list']);

// Supprimer un item précis
\Drupal::cache('data')->delete($cid);

// Supprimer plusieurs items
\Drupal::cache('data')->deleteMultiple([$cid1, $cid2]);
```

---

## Désactiver le Cache en Développement

```php
// web/sites/default/settings.local.php

// 1. Null backends — pas de mise en cache du tout
$settings['cache']['bins']['render'] = 'cache.backend.null';
$settings['cache']['bins']['page'] = 'cache.backend.null';
$settings['cache']['bins']['dynamic_page_cache'] = 'cache.backend.null';

// 2. Twig auto-reload
$settings['twig.config']['debug'] = TRUE;
$settings['twig.config']['auto_reload'] = TRUE;
$settings['twig.config']['cache'] = FALSE;

// 3. Afficher les erreurs PHP
$config['system.logging']['error_level'] = 'verbose';

// 4. Désactiver CSS/JS aggregation
$config['system.performance']['css']['preprocess'] = FALSE;
$config['system.performance']['js']['preprocess'] = FALSE;
```

```yaml
# web/sites/default/services.yml (pour Twig debug depuis le container)
parameters:
  twig.config:
    debug: true
    auto_reload: true
    cache: false
```

---

## Configuration des Backends (settings.php)

```php
// Utiliser une base de données différente pour le cache
$settings['cache']['default'] = 'cache.backend.database';

// Redis comme backend pour tous les bins (voir redis-memcache.md)
$settings['cache']['default'] = 'cache.backend.redis';

// Mélanger les backends selon le bin
$settings['cache']['bins']['render'] = 'cache.backend.redis';
$settings['cache']['bins']['default'] = 'cache.backend.database';
$settings['cache']['bins']['bootstrap'] = 'cache.backend.chainedfast';
```

---

## Debug du Cache

```bash
# Vérifier l'état du cache
drush php:eval "echo \Drupal::cache()->get('test') ? 'HIT' : 'MISS';"

# Invalider tous les caches (à éviter en production)
drush cache:rebuild

# Invalider par tags depuis Drush
drush php:eval "\Drupal\Core\Cache\Cache::invalidateTags(['node_list']);"

# Vider un bin spécifique
drush php:eval "\Drupal::cache('render')->deleteAll();"

# Vérifier le module Page Cache
drush php:eval "echo \Drupal::moduleHandler()->moduleExists('page_cache') ? 'actif' : 'inactif';"

# Afficher les headers de cache d'une page (depuis l'extérieur)
curl -I https://mon-site.com/node/1 | grep -i "X-Drupal-Cache\|Cache-Control\|X-Cache"
```
