---
name: drupal-performance — cache tags et contexts
description: Guide complet sur les cache tags (invalidation granulaire), cache contexts (variation du cache), et cache max-age. Patterns corrects pour les render arrays, entités, listes, et composants custom.
---

# Cache Tags & Cache Contexts — Guide Complet

## Comprendre le Modèle

```
Cache Tags  → QUOI invalide le cache (liés aux DONNÉES)
             "Ce render array dépend du nœud 42 et du config system.site"
             Quand node:42 change → tout ce qui a ce tag est invalidé

Cache Contexts → POUR QUI/QUOI varier le cache (liés à la DIMENSION)
                "Ce bloc est différent selon la langue et le rôle utilisateur"
                Drupal stocke N versions du bloc selon les contexts actifs

Cache Max-Age → COMBIEN DE TEMPS garder ce cache (TTL en secondes)
               0 = jamais mis en cache
               -1 = Cache::PERMANENT (invalider par tags uniquement)
               3600 = 1 heure
```

---

## Cache Tags — Référence Complète

### Tags d'entités natifs Drupal

```php
// Un nœud spécifique
['node:42']

// Tous les nœuds (liste) — invalider quand un nœud est ajouté/supprimé
['node_list']

// Nœuds d'un type de contenu
['node_list:article']

// Un utilisateur spécifique
['user:1']

// Liste d'utilisateurs
['user_list']

// Un terme de taxonomy
['taxonomy_term:15']

// Une config spécifique
['config:system.site']
['config:node.settings']

// Tout le cache config
['config:*']   // À éviter — trop large

// Un block spécifique
['block:mon_block']

// Une View spécifique
['config:views.view.ma_view']

// Toutes les vues
['views_data']
```

### Obtenir les tags d'une entité

```php
// Tags d'un nœud chargé
$node = Node::load(42);
$tags = $node->getCacheTags();
// → ['node:42']

// Tags pour la liste d'entités d'un type
$tags = \Drupal::entityTypeManager()
  ->getDefinition('node')
  ->getListCacheTags();
// → ['node_list']

// Merger des tags
use Drupal\Core\Cache\Cache;
$tags = Cache::mergeTags($node->getCacheTags(), $user->getCacheTags());
// → ['node:42', 'user:1']
```

### Invalider des Tags

```php
// Invalider depuis PHP (triggère la purge de toutes les caches avec ce tag)
Cache::invalidateTags(['node:42']);
Cache::invalidateTags(['node_list', 'user_list']);

// Drupal invalide automatiquement lors d'un save() ou delete()
$node->save();     // → invalide automatiquement node:42 + node_list
$node->delete();   // → invalide automatiquement node:42 + node_list
```

### Tags custom dans un render array

```php
$build = [
  '#theme' => 'mon_composant',
  '#data' => $data,
  '#cache' => [
    'tags' => [
      'mon_module_donnees',           // Tag custom — à invalider manuellement
      'node:' . $node->id(),          // Invalider si ce nœud change
      'config:mon_module.settings',   // Invalider si la config change
    ],
    'max-age' => Cache::PERMANENT,
  ],
];

// Plus tard, invalider ce tag custom
Cache::invalidateTags(['mon_module_donnees']);
```

---

## Cache Contexts — Référence Complète

### Contexts natifs Drupal

```php
// Contexts disponibles dans Drupal core :

// URL et navigation
'url'                    // URL complète (très granulaire)
'url.path'               // Chemin URL sans query string
'url.query_args'         // Tous les query parameters
'url.query_args:page'    // Query param spécifique
'url.path.is_front'      // Est-ce la page d'accueil ?

// Utilisateur
'user'                   // Utilisateur courant (très granulaire)
'user.roles'             // Rôles de l'utilisateur (moins granulaire)
'user.permissions'       // Permissions individuelles (entre user et user.roles)

// Langue
'languages'                          // Toutes les langues (language type)
'languages:language_interface'       // Langue de l'interface UI
'languages:language_content'         // Langue du contenu
'languages:language_url'             // Langue détectée depuis l'URL

// Session
'session'                // Session PHP complète
'session.exists'         // Session existe ou non (boolean)

// Route
'route'                  // Route courante (nom + paramètres)
'route.name'             // Nom de la route uniquement

// Headers HTTP
'headers:Accept-Language'  // Header Accept-Language
```

### Choisir le Context le Plus Précis

```php
// ❌ Trop granulaire — chaque utilisateur a sa propre version du cache
'#cache' => ['contexts' => ['user']],

// ✅ Mieux — partager le cache par rôle
'#cache' => ['contexts' => ['user.roles']],

// ✅ Encore mieux si le contenu dépend d'une permission précise
'#cache' => ['contexts' => ['user.permissions']],

// ❌ Trop granulaire pour un composant commun à toutes les pages
'#cache' => ['contexts' => ['url']],

// ✅ Si seulement le chemin varie
'#cache' => ['contexts' => ['url.path']],

// ✅ Si seulement la pagination varie
'#cache' => ['contexts' => ['url.query_args:page']],
```

### Définir un Cache Context Custom

```php
// src/Cache/Context/CommandeStatutCacheContext.php
namespace Drupal\mon_module\Cache\Context;

use Drupal\Core\Cache\Context\CacheContextInterface;
use Drupal\Core\Cache\CacheableMetadata;

class CommandeStatutCacheContext implements CacheContextInterface {

  public static function getLabel(): \Stringable {
    return t('Statut des commandes de l\'utilisateur courant');
  }

  public function getContext(): string {
    // Valeur utilisée pour différencier les versions du cache
    $user_id = \Drupal::currentUser()->id();
    $statut = /* logique pour obtenir le statut */ 'confirmed';
    return $user_id . ':' . $statut;
  }

  public function getCacheableMetadata(): CacheableMetadata {
    return new CacheableMetadata();
  }
}
```

```yaml
# mon_module.services.yml
services:
  cache_context.mon_module.commande_statut:
    class: Drupal\mon_module\Cache\Context\CommandeStatutCacheContext
    tags:
      - { name: cache.context }
```

```php
// Utilisation dans un render array
$build['#cache']['contexts'][] = 'mon_module.commande_statut';
```

---

## Patterns Complets par Cas d'Usage

### Render Array d'un Nœud

```php
$node = Node::load($nid);
$build = [
  '#theme' => 'node',
  '#node' => $node,
  '#view_mode' => 'full',
  '#langcode' => \Drupal::languageManager()->getCurrentLanguage()->getId(),
  '#cache' => [
    'keys' => ['node', $nid, 'full'],             // Clé de cache (pour les composants)
    'tags' => $node->getCacheTags(),              // ['node:42']
    'contexts' => ['languages:language_content'],  // Varie par langue du contenu
    'max-age' => Cache::PERMANENT,
  ],
];
```

### Bloc Personnalisé selon le Rôle

```php
// BlockBase::build() — cache correct pour un bloc selon le rôle
return [
  '#markup' => $this->t('Contenu pour @role', ['@role' => $role]),
  '#cache' => [
    'contexts' => ['user.roles'],    // Une version par combinaison de rôles
    'tags' => ['config:user.role.editor'],
    'max-age' => Cache::PERMANENT,
  ],
];
```

### Liste Paginée de Nœuds

```php
$build = [
  '#theme' => 'item_list',
  '#items' => $items,
  '#cache' => [
    'tags' => ['node_list'],                          // Invalider si un nœud change
    'contexts' => ['url.query_args:page', 'languages'], // Varie par page et langue
    'max-age' => 3600,                                // TTL 1 heure en plus des tags
  ],
];
```

### Composant avec Données Externes (API)

```php
// Contenu chargé depuis une API externe — pas de tags Drupal
$build = [
  '#markup' => $api_data,
  '#cache' => [
    'max-age' => 3600,          // TTL : revérifier l'API toutes les heures
    'contexts' => ['url.path'], // Différent par page
    // Pas de tags — l'invalidation est purement basée sur le TTL
  ],
];
```

### Composant Jamais Mis en Cache

```php
// Bloc de connexion / logout — toujours frais
return [
  '#markup' => $this->buildAuthBlock(),
  '#cache' => [
    'max-age' => 0,   // JAMAIS mis en cache — recalculé à chaque requête
  ],
];
```

---

## Debugging du Cache

```bash
# Vérifier les tags envoyés dans les headers HTTP (si configuré)
curl -I https://mon-site.com/node/1 | grep X-Drupal-Cache-Tags

# En PHP — afficher les tags d'un render array après rendu
$renderer = \Drupal::service('renderer');
$metadata = $renderer->renderRoot($build);
// Les cache tags sont maintenant dans $build['#cache']['tags']

# Vérifier les contexts résolus d'une page
drush php:eval "
\$contexts = \Drupal::service('cache_contexts_manager');
\$keys = \$contexts->convertTokensToKeys(['user.roles', 'languages:language_interface'])->getKeys();
var_dump(\$keys);
"
```
