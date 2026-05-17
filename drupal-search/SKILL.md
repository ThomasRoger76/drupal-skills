---
name: drupal-search
description: Use when implementing site search with Search API module, configuring search indexes (fields to index, processors), connecting to Solr backend (solr module, schema.xml, field mapping), connecting to Elasticsearch or OpenSearch backend, building search pages with Views + Search API, implementing faceted search with the Facets module, creating custom Search API processors for data transformation or query alteration, configuring multilingual search (langcode, language-specific analyzers), debugging Search API indexing issues (drush sapi-i, sapi-c), implementing search autocomplete, configuring full-text search relevance scoring, or building search pages without Views (SearchApiQuery API) in Drupal 8-11+
---

# Drupal Search — Search API & Full-Text Search — Référence Complète

## Overview

Référentiel complet de Search API Drupal 8-11+ : configuration des indexes, backends (Database, Solr, Elasticsearch/OpenSearch), Views + Search API, Facets, processors custom PHP, multilingual search, autocomplete, debugging. Search API est le standard de facto — Solr est le backend dominant en production agence française (4/20 projets). Facets module est utilisé dans 60%+ des projets.

> **Fichiers de référence avec code PHP** : processors custom complets dans [custom-processors.md](custom-processors.md), config Solr détaillée dans [solr-integration.md](solr-integration.md), facettes dépendantes dans [facets.md](facets.md).

## 🎯 La Règle Fondamentale

> **Search API, pas le module `search` core.** Le module `search` core est limité à une recherche plein texte basique sur les nœuds. Search API indexe n'importe quelle entité, supporte des backends scalables (Solr, Elasticsearch), s'intègre nativement à Views, et supporte les facettes.

---

## Choisir le Backend

| Backend | Cas d'usage | Avantages | Inconvénients |
|---------|-------------|-----------|---------------|
| **Database** (core) | Dev local, petits sites (<5000 items) | 0 infra | Pas de relevance scoring, lent |
| **Solr** | Sites de taille moyenne à grande | Relevance, facettes, multilingue | Infra Solr nécessaire |
| **Elasticsearch/OpenSearch** | Sites très large échelle, analytics | Scalabilité, aggregations | Configuration complexe |

**Règle :** Database en développement, Solr en production pour tout site > 5000 entités indexées.

---

## Quick Decision Table

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Créer un index Search API | `/admin/config/search/search-api/add-index` | [search-api-setup.md](search-api-setup.md) |
| Choisir les champs à indexer | Index → Fields tab — ajouter les champs pertinents | [search-api-setup.md](search-api-setup.md) |
| Indexer les champs des entités référencées | Index → Fields → Add related fields | [search-api-setup.md](search-api-setup.md) |
| Configurer les processors (HTML filter, Tokenizer) | Index → Processors tab | [search-api-setup.md](search-api-setup.md) |
| Lancer l'indexation complète | `drush sapi-i` (search-api:index) | [search-api-setup.md](search-api-setup.md) |
| Vider et réindexer | `drush sapi-c && drush sapi-i` | [search-api-setup.md](search-api-setup.md) |
| Statut de l'indexation | `drush sapi-s` (search-api:status) | [search-api-setup.md](search-api-setup.md) |
| Connecter un backend Solr | Module `search_api_solr` + configurer le server | [solr-integration.md](solr-integration.md) |
| Configurer Solr dans Docker | Service Solr + core Drupal spécifique | [solr-integration.md](solr-integration.md) |
| Générer le schéma Solr depuis Drupal | `drush solr-gsc` (generate-solr-config) | [solr-integration.md](solr-integration.md) |
| Connecter Elasticsearch | Module `elasticsearch_connector` | [elasticsearch.md](elasticsearch.md) |
| Page de recherche avec résultats | Views — ajouter une "Search API Index" View | [search-views.md](search-views.md) |
| Exposer la recherche plein texte | Views → Exposed filter → Search API Fulltext | [search-views.md](search-views.md) |
| Filtrer par content type dans Views search | Views → filter sur `type` | [search-views.md](search-views.md) |
| Facettes (filtres à cocher) | Module `facets` + Views + Search API | [facets.md](facets.md) |
| Facette sur taxonomy term | Facets → field `field_tags` → term facet | [facets.md](facets.md) |
| Facette avec count (32) à côté | Facets → afficher le count de résultats | [facets.md](facets.md) |
| Facettes dépendantes (hiérarchiques) | Facets — URL processors + Dependent Facets | [facets.md](facets.md) |
| Transformer les données à l'indexation | Custom DataAlterCallback processor | [custom-processors.md](custom-processors.md) |
| Modifier la query Search API | Custom SearchApiQuery hook ou QueryPreprocessorInterface | [custom-processors.md](custom-processors.md) |
| Boost de relevance par champ | Solr → field boost, ou Search API field weight | [solr-integration.md](solr-integration.md) |
| Recherche multilingue (analyseurs par langue) | Search API Solr + Language Fallback processor | [solr-integration.md](solr-integration.md) |
| Autocomplete dans le champ de recherche | `drupal/search_api_autocomplete` module | [search-api-setup.md](search-api-setup.md) |
| Highlighting des termes recherchés | Solr → Excerpt processor (HTML avec balises <em>) | [solr-integration.md](solr-integration.md) |
| Recherche phonétique (soundex, metaphone) | Solr — phonetic field type dans schema | [solr-integration.md](solr-integration.md) |
| Search API sans Views (API PHP) | `Index::query()` + `QueryInterface` | [custom-processors.md](custom-processors.md) |
| **OpenSearch / AWS OpenSearch backend** | `drupal/elasticsearch_connector` ≥ 1.x — supporte Elasticsearch ET OpenSearch via la même API | [elasticsearch.md](elasticsearch.md) |
| **Tolérance aux fautes de frappe (fuzzy search)** | Solr — `lowercaseFilter` + `phonetic` OU `edgeNGram` tokenizer | [solr-integration.md](solr-integration.md) |
| **Tracking des requêtes de recherche** | `drupal/search_api_stats` — logs et rapports des termes recherchés | [search-api-setup.md](search-api-setup.md) |
| **Recherche "did you mean" (suggestions)** | Solr SpellCheck component — configurer dans `solrconfig.xml` + Search API Solr settings → "Spellcheck" | [solr-integration.md](solr-integration.md) |
| **Synonymes dans la recherche** | Solr synonym.txt dans la config du schema | [solr-integration.md](solr-integration.md) |
| **Réindexer automatiquement après mise à jour** | `drupal/search_api_solr` → Index automatique via Queue Worker à la sauvegarde d'entité | [search-api-setup.md](search-api-setup.md) |
| **Filtrer les résultats par permissions Drupal** | Search API Processor `Content Access` — obligatoire sur tout site avec accès restreint | [search-api-setup.md](search-api-setup.md) |
| **Search API avec filtres géographiques (geosearch)** | `drupal/geofield` + Search API champ `geofield` + Solr RPT handler | [custom-processors.md](custom-processors.md) |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| Module `search` core pour des besoins avancés | Search API + backend approprié | Pas de facettes, relevance, multilingue |
| Indexer TOUS les champs sans réflexion | Sélectionner uniquement les champs utiles à la recherche | Index trop lourd, indexation lente |
| Backend Database en production (> 5000 entités) | Solr ou Elasticsearch | Requêtes lentes, pas de relevance scoring |
| `drush sapi-i` sans `drush sapi-c` après changement de schéma | Vider l'index avant de réindexer | Données obsolètes dans l'index |
| Schema Solr généré manuellement | `drush solr-gsc` puis copier dans Solr | Schema incorrect, indexation partielle |
| Views Search sans pagination | Toujours paginer — 10 ou 20 résultats | Pages très lentes, timeout |
| Facette sans filtre "0 results" | Configurer "Min count = 1" dans Facets | Facettes vides affichées |
| Indexation des révisions non publiées | Processor `Content Access` pour filtrer | Contenu draft visible dans les résultats |
| Cron désactivé pour l'indexation auto | Configurer le cron Drupal (drush cron) | Nouveaux contenus non indexés |
| Recherche sans Content Access processor | Toujours activer `Content Access` | Contenu privé visible dans les résultats |

## Évolution par Version Majeure

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| Search API (contrib) | ✅ | ✅ | ✅ | ✅ |
| Search API Solr (contrib) | ✅ | ✅ | ✅ | ✅ |
| Facets module (contrib) | ✅ | ✅ | ✅ | ✅ |
| Search API Autocomplete | ✅ | ✅ | ✅ | ✅ |
| Elasticsearch Connector | ✅ | ✅ | ✅ | ✅ |
| `drush sapi-i/c/s` | ✅ | ✅ | ✅ | ✅ |
| Views + Search API | ✅ | ✅ | ✅ | ✅ |
| Search API Solr 4.x (D10+) | ❌ | ❌ | ✅ | ✅ |
| Database backend (core Search API) | ✅ | ✅ | ✅ | ✅ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Problèmes d'indexation et de configuration rencontrés en projet.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions.

## See Also

- `drupal-views` — Views + Search API, exposed filters search, facets dans Views
- `drupal-docker` — Service Solr Docker, Elasticsearch Docker
- `drupal-performance` — Cache des résultats de recherche, Varnish + Search
- `drupal-multilingual` — Recherche multilingue, analyseurs par langue dans Solr
- `drupal-config` — Exporter/importer la config Search API, indexes, servers
- `drupal-core` — Cron (indexation auto), hook_search_api_query_alter()
