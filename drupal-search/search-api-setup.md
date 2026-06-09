---
name: drupal-search — search api setup
description: Configuration complète de Search API Drupal - création d'un index, choix des champs, processors, commandes drush, backends, et intégration avec Views.
---

# Search API — Configuration & Setup Complet

## Installation et Prérequis

```bash
# Search API + backend Database (dev)
composer require drupal/search_api

# Search API + Solr (production)
composer require drupal/search_api_solr

# Facettes
composer require drupal/facets

# Autocomplete
composer require drupal/search_api_autocomplete

# Activer les modules
drush en search_api search_api_db -y          # Database backend
drush en search_api_solr -y                   # Solr backend (si Solr)
drush en facets -y                            # Facettes
```

---

## Créer un Serveur Search API

### Serveur Database (Développement)

```yaml
# config/install/search_api.server.database_server.yml
langcode: fr
status: true
id: database_server
name: 'Database Server (Dev)'
description: 'Backend base de données — uniquement pour le développement.'
backend: search_api_db
backend_config:
  min_chars: 3
  partial_matches: false
  matching: words
  autocomplete:
    suggest_suffix: true
    suggest_words: true
```

### Serveur Solr (Production)

```yaml
# config/install/search_api.server.solr_server.yml
langcode: fr
status: true
id: solr_server
name: 'Solr Server'
description: 'Backend Solr pour la production.'
backend: search_api_solr
backend_config:
  connector: standard
  connector_config:
    scheme: http
    host: solr
    port: '8983'
    path: '/'
    core: drupal
    timeout: 5
    index_timeout: 5
    query_timeout: 5
    finalize_timeout: 30
  retrieve_data: false
  highlight_data: false
  skip_schema_check: false
  solr_version: ''
  http_method: AUTO
  commit_within: 1000
  jmx_enabled: false
  solr_install_dir: ''
```

---

## Créer un Index

```yaml
# config/install/search_api.index.articles_index.yml
langcode: fr
status: true
id: articles_index
name: 'Index des Articles'
description: 'Index de recherche pour les articles du site.'
read_only: false
field_settings:
  title:
    label: Titre
    datasource_id: 'entity:node'
    property_path: title
    type: text
    boost: '21.0'            # ← Boost fort sur le titre
  body:
    label: 'Corps de l\'article'
    datasource_id: 'entity:node'
    property_path: body
    type: text
    boost: '1.0'
  field_tags_name:
    label: 'Nom du tag'
    datasource_id: 'entity:node'
    property_path: 'field_tags:entity:name'    # ← Champ de relation
    type: string
  status:
    label: Statut
    datasource_id: 'entity:node'
    property_path: status
    type: boolean
  created:
    label: 'Date de création'
    datasource_id: 'entity:node'
    property_path: created
    type: date
  langcode:
    label: Langue
    datasource_id: 'entity:node'
    property_path: langcode
    type: string
  type:
    label: 'Type de contenu'
    datasource_id: 'entity:node'
    property_path: type
    type: string
datasource_settings:
  'entity:node':
    plugin_id: 'entity:node'
    settings:
      bundles:
        default: true
        selected:
          - article
          - page
      languages:
        default: true
        selected: []
tracker_settings:
  default:
    plugin_id: default
    settings: {}
processor_settings:
  add_url:
    processor_id: add_url
    weights:
      preprocess_index: -20
    settings: {}
  content_access:
    processor_id: content_access
    weights:
      preprocess_index: -10
      preprocess_query: -10
    settings: {}
  html_filter:
    processor_id: html_filter
    weights:
      preprocess_index: -15
    settings:
      all_fields: false
      fields:
        - body
      title: false
      alt: true
      tags:
        h1: '5'
        h2: '3'
        h3: '2'
        strong: '2'
        b: '2'
  ignorecase:
    processor_id: ignorecase
    weights:
      preprocess_index: -20
      preprocess_query: -20
    settings:
      all_fields: true
  tokenizer:
    processor_id: tokenizer
    weights:
      preprocess_index: -25
      preprocess_query: -25
    settings:
      all_fields: true
      spaces: ''
      punctuation: ''
  stopwords:
    processor_id: stopwords
    weights:
      preprocess_index: -10
      preprocess_query: -10
    settings:
      all_fields: true
      stopwords: "de\ndu\nla\nle\nles\net\nen\nun\nune\ndes\nà\nau"
server: solr_server
options:
  index_directly: true
  cron_limit: '50'
```

---

## Commandes Drush Search API

```bash
# Statut de tous les index
drush search-api:status
drush sapi-s

# Indexer les éléments en attente
drush search-api:index articles_index
drush sapi-i articles_index

# Vider un index (désindexer)
drush search-api:clear articles_index
drush sapi-c articles_index

# Vider et réindexer
drush sapi-c articles_index && drush sapi-i articles_index

# Activer un index
drush search-api:enable articles_index

# Désactiver un index
drush search-api:disable articles_index

# Lancer le cron Search API (indexation batch)
drush search-api:run-tasks

# Voir les éléments en attente d'indexation
drush php:eval "
\$index = \Drupal::entityTypeManager()->getStorage('search_api_index')->load('articles_index');
if (\$index) {
  echo 'Items restants: ' . \$index->getTrackerInstance()->getRemainingItemsCount() . PHP_EOL;
  echo 'Items indexés: ' . \$index->getTrackerInstance()->getIndexedItemsCount() . PHP_EOL;
}
"

# Pour Solr — générer le schéma
drush search-api-solr:generate-solr-config articles_index /tmp/solr-config/
# (ou drush solr-gsc si Drush Solr integration module)
```

---

## Processors Disponibles — Référence

| Processor | Usage | Quand activer |
|-----------|-------|---------------|
| `add_url` | Ajoute l'URL de l'entité à l'index | Toujours |
| `content_access` | Filtre selon les permissions Drupal | Toujours — sécurité |
| `html_filter` | Nettoie le HTML avant indexation | Si du HTML est indexé |
| `tokenizer` | Découpe le texte en tokens | Toujours pour la recherche full-text |
| `ignorecase` | Rend la recherche insensible à la casse | Toujours |
| `stopwords` | Ignore les mots vides (de, la, le...) | Recommandé |
| `stemmer` | Réduit les mots à leur racine | Pour une meilleure pertinence |
| `language_with_fallback` | Gestion du fallback multilingue | Sites multilingues |
| `highlight` | Surligne les termes dans les résultats | Si Solr avec highlight |
| `reverse_entity_references` | Index les entités qui référencent cette entité | Cas avancés |
| `rendered_item` | Indexe le HTML rendu | Cas très avancés |

---

## Intégration Views + Search API

```yaml
# Une View utilisant l'index Search API comme source de données
# config/install/views.view.recherche.yml
base_table: search_api_index_articles_index
base_field: search_api_id

# Dans l'UI Views :
# 1. Créer une View → choisir "Index: Index des Articles" comme source
# 2. Ajouter un champ "Titre" (vient de l'index, pas de node_field_data)
# 3. Ajouter un filtre exposé "Recherche plein texte Search API"
# 4. Ajouter les filtres sur les facettes (type, tags, date)
# 5. Configurer le tri (pertinence, date)
```

**Champ "Recherche plein texte" dans Views :**
- Views → Filtres → Ajouter → Chercher "Fulltext search"
- Cocher "Expose this filter to visitors"
- L'éditeur peut configurer les champs recherchés (title seulement vs title+body)

---

## Indexation Automatique et Cron

```php
// L'indexation automatique est configurée dans l'index :
// 'index_directly: true' → indexe immédiatement lors du save()
// 'cron_limit: 50' → max 50 items par exécution de cron

// Pour forcer l'indexation d'un item spécifique :
$index = \Drupal::entityTypeManager()
  ->getStorage('search_api_index')
  ->load('articles_index');

$datasource_id = 'entity:node';
// L'identifiant de tracking inclut le langcode : "entity:node/{nid}:{langcode}".
$tracking_id = $datasource_id . '/42:fr';

// trackItemsUpdated attend la liste des IDs de tracking (avec langcode).
$index->trackItemsUpdated($datasource_id, [$tracking_id]);

// Pour déclencher l'indexation en batch (sans attendre le cron) :
$index->indexItems(100);  // indexer 100 items

// Hook pour ré-indexer quand une entité référencée change :
use Drupal\Core\Entity\EntityInterface;
use Drupal\taxonomy\TermInterface;

function mon_module_entity_update(EntityInterface $entity): void {
  if (!$entity instanceof TermInterface) {
    return;
  }

  // Réindexer tous les nœuds qui référencent ce terme.
  $nids = \Drupal::entityQuery('node')
    ->condition('field_tags', $entity->id())
    ->accessCheck(FALSE)
    ->execute();

  if (!$nids) {
    return;
  }

  $index = \Drupal::entityTypeManager()
    ->getStorage('search_api_index')
    ->load('articles_index');

  // Marquer les nœuds comme à réindexer. Search API résout les langcodes
  // de chaque entité ; passer les nids suffit via trackItemsUpdated sur la datasource.
  $tracking_ids = array_map(static fn (string $nid): string => "$nid:" . $entity->language()->getId(), $nids);
  $index->trackItemsUpdated('entity:node', $tracking_ids);
}
```
