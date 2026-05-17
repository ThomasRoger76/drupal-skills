---
name: drupal-views — REST export
description: Configurer un display REST Export dans Views pour exposer des données en JSON/XML/CSV, configurer l'authentification, intégrer avec Search API, et optimiser le cache des exports.
---

# Views REST Export — Référence Complète

## Créer un Display REST Export

```
Views → Add display → REST export

Configuration :
├── Path settings → Path : /api/mes-articles    (sans _format)
│     → Drupal ajoute automatiquement ?_format=json
├── Authentication → Basic Auth / Session / OAuth
├── Format → Serializer (JSON, XML selon les modules)
└── Show → Fields ou Entity (entité complète normalisée)
```

**Accès :**
```bash
# Format JSON
GET /api/mes-articles?_format=json

# Format XML (si serialization_xml est activé)
GET /api/mes-articles?_format=xml

# Format CSV (avec module csv_serialization)
GET /api/mes-articles?_format=csv
```

---

## Configuration JSON — Formats de Réponse

### Show: Fields (JSON d'objets plats)

```json
[
  {
    "nid": "42",
    "title": "Mon article",
    "body": "<p>Contenu...</p>",
    "created": "1716825600"
  },
  {
    "nid": "43",
    "title": "Autre article",
    "body": "<p>Autre contenu...</p>",
    "created": "1716912000"
  }
]
```

**Configuration :** Ajouter uniquement les champs nécessaires — chaque champ Views = une clé JSON.

**Renommer les clés JSON :** Views → Champ → Alias (le label du champ = clé JSON si pas d'alias).

### Show: Entity (JSON normalisé complet)

```json
[
  {
    "nid": [{"value": 42}],
    "uuid": [{"value": "abc-123-def"}],
    "title": [{"value": "Mon article"}],
    "status": [{"value": true}],
    "created": [{"value": 1716825600, "format": "Y-m-d\\TH:i:sP", "value": "2026-05-27T..."}]
  }
]
```

**Recommandation :** Utiliser "Fields" pour un format propre et maîtrisé. "Entity" pour la migration de données.

---

## Authentification sur les REST Export

```
Display → Authentication → Change

Options :
├── Anonymous (pas d'authentification)
├── Basic Auth (user:password en base64)
├── Cookie (session Drupal — pour les frontends intégrés)
└── OAuth (avec simple_oauth module)
```

**Sécuriser un endpoint :**
```
Access → Permission → "Access GET on Article resource"
# OU
Access → Custom (voir custom-handlers.md)
```

---

## Configurer les Filtres Exposés via URL

```bash
# Filtre exposé 'type' — filtrer par content type
GET /api/contenu?type=article&_format=json

# Filtre exposé 'langcode' — filtrer par langue
GET /api/contenu?langcode=fr&_format=json

# Combinaison de filtres
GET /api/contenu?type=article&langcode=fr&sort_by=created&sort_order=DESC&_format=json

# Pagination
GET /api/contenu?_format=json&page=0&items_per_page=20
```

---

## Ajouter des Métadonnées à la Réponse (Header / Footer)

```php
// hook_views_rest_api_response_alter — modifier la réponse REST Views
function mon_module_views_rest_api_response_alter(array &$response, \Drupal\views\ViewExecutable $view): void {
  if ($view->id() !== 'mes_articles_api') {
    return;
  }

  // Ajouter des métadonnées au niveau racine
  // ⚠️ Format Views REST = tableau de résultats — wrapper nécessaire
}

// Alternativement : créer une réponse JSON structurée via un RestResource custom
// src/Plugin/rest/resource/ArticlesResource.php
```

---

## REST Resource Custom (Alternative aux Views REST Export)

Pour les endpoints avec logique métier complexe, un `RestResource` plugin est plus approprié :

```php
<?php
// src/Plugin/rest/resource/ArticlesResource.php
namespace Drupal\mon_module\Plugin\rest\resource;

use Drupal\rest\Plugin\ResourceBase;
use Drupal\rest\ResourceResponse;
use Symfony\Component\HttpKernel\Exception\BadRequestHttpException;

/**
 * @RestResource(
 *   id = "mon_module_articles",
 *   label = @Translation("Articles API"),
 *   uri_paths = {
 *     "canonical" = "/api/v1/articles/{id}",
 *     "create" = "/api/v1/articles",
 *   }
 * )
 */
// D11 : #[RestResource(...)] attribute
class ArticlesResource extends ResourceBase {

  public function get(int $id): ResourceResponse {
    $node = \Drupal::entityTypeManager()->getStorage('node')->load($id);

    if (!$node || $node->bundle() !== 'article') {
      throw new \Symfony\Component\HttpKernel\Exception\NotFoundHttpException();
    }

    if (!$node->access('view')) {
      throw new \Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException();
    }

    $data = [
      'id' => $node->id(),
      'title' => $node->getTitle(),
      'body' => $node->get('body')->value,
      'created' => $node->getCreatedTime(),
    ];

    $response = new ResourceResponse($data, 200);
    $response->addCacheableDependency($node);

    return $response;
  }
}
```

**Activer le REST Resource :**
```yaml
# config/install/rest.resource.mon_module_articles.yml
langcode: fr
status: true
id: mon_module_articles
plugin_id: mon_module_articles
granularity: resource
configuration:
  methods:
    - GET
    - POST
  formats:
    - json
  authentication:
    - basic_auth
    - cookie
```

---

## Views REST Export + Search API

Pour exposer des résultats de recherche en JSON :

```
1. Créer une View avec source "Index: Mon Index"
2. Add display → REST export
3. Path : /api/recherche
4. Show → Fields (ajouter : titre, résumé, URL, image)
5. Filter → Fulltext search (exposed → identifier: "q")
6. Format → Serializer → JSON

# Appel :
GET /api/recherche?q=drupal&_format=json
GET /api/recherche?q=drupal&field_tags=42&_format=json
```

---

## Cache des Views REST Export

```php
// hook_views_pre_render — ajouter des cache tags à l'export REST
function mon_module_views_pre_render(\Drupal\views\ViewExecutable $view): void {
  if ($view->id() !== 'articles_api') {
    return;
  }

  // Cache valide jusqu'à ce qu'un article change
  $view->element['#cache']['tags'][] = 'node_list:article';
  $view->element['#cache']['max-age'] = 3600;  // 1 heure max
}
```

**Headers de cache dans la réponse REST :**
```bash
curl -I /api/mes-articles?_format=json
# X-Drupal-Cache-Tags: node_list node:42 node:43
# Cache-Control: max-age=3600, public
```

---

## Format CSV avec csv_serialization

```bash
composer require drupal/csv_serialization
drush en csv_serialization -y

# Accès
GET /api/export?_format=csv

# Headers de réponse CSV
Content-Type: text/csv; charset=UTF-8
Content-Disposition: attachment; filename="export.csv"
```

**Configuration :** Views REST export → Format → Serializer → configurer `csv` comme format disponible.
