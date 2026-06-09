---
name: drupal-search — elasticsearch
description: Intégration Elasticsearch / OpenSearch avec Search API Drupal - configuration du connector, mapping des champs, et comparaison avec Solr.
---

# Elasticsearch & OpenSearch — Intégration Search API

## Quand Choisir Elasticsearch vs Solr

| Critère | Elasticsearch/OpenSearch | Solr |
|---------|--------------------------|------|
| Scalabilité | ✅ Natif (shards/répliques) | ✅ SolrCloud |
| Analytics/Aggregations | ✅ Très puissant | ⚠️ Limité |
| Configuration initiale | ⚠️ Plus complexe | ✅ Plus simple avec Drupal |
| Module Drupal stable | ⚠️ Moins mature | ✅ Très mature |
| Recommandé pour Drupal | Sites très large échelle | Sites moyens-grands |

**Recommandation :** Utiliser Solr pour les projets Drupal standard. Elasticsearch pour les besoins d'analytics avancés ou les équipes déjà familières avec la stack ELK.

---

## Installation

Deux modules **concurrents** (jamais ensemble — choisir l'un OU l'autre) :

| Module | Backend Search API | Notes |
|--------|--------------------|-------|
| `drupal/elasticsearch_connector` | `elasticsearch` (8.x réécrit pour ES 7/8 + OpenSearch) | Le plus utilisé, supporte ES et OpenSearch via la même API |
| `drupal/search_api_opensearch` | `opensearch` | Fork orienté AWS OpenSearch, maintenu activement |

```bash
# Option A — Elasticsearch Connector (Elasticsearch OU OpenSearch)
composer require drupal/elasticsearch_connector
drush en elasticsearch_connector -y

# Option B — Search API OpenSearch (AWS OpenSearch)
composer require drupal/search_api_opensearch
drush en search_api_opensearch -y
```

> Ne pas activer les deux : ils fournissent des backends distincts et entrent en conflit sur le mapping des champs.

---

## Service Elasticsearch dans Docker Compose

```yaml
# docker-compose.yml
services:
  elasticsearch:
    image: elasticsearch:8.13-slim
    ports:
      - "9200:9200"
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false    # Désactiver en dev uniquement
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
      - cluster.name=drupal-search
    volumes:
      - es_data:/usr/share/elasticsearch/data
    ulimits:
      memlock:
        soft: -1
        hard: -1

  # OpenSearch (alternative open-source)
  opensearch:
    image: opensearchproject/opensearch:2
    ports:
      - "9200:9200"
    environment:
      - discovery.type=single-node
      - DISABLE_SECURITY_PLUGIN=true    # Dev uniquement
      - OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m
    volumes:
      - os_data:/usr/share/opensearch/data

volumes:
  es_data:
  os_data:
```

---

## Configuration du Serveur

`elasticsearch_connector` 8.x ne définit plus d'entité « cluster » séparée :
la connexion est portée directement par le serveur Search API.

```yaml
# config/install/search_api.server.elasticsearch_server.yml
langcode: fr
status: true
id: elasticsearch_server
name: 'Elasticsearch Server'
description: ''
backend: elasticsearch
backend_config:
  connector: standard
  connector_config:
    url: 'http://elasticsearch:9200'   # URL du service Docker
    enable_debug_logging: false
    username: ''                       # Auth (xpack security) si activée
    password: ''
  fuzziness: 'auto'                    # Recherche approximative (tolérance fautes)
  advanced:
    prefix: ''
    suffix: ''
```

> Pour OpenSearch via `drupal/search_api_opensearch`, le backend devient
> `opensearch` et la clé `connector_config.url` pointe vers le endpoint OpenSearch
> (avec auth IAM/basic selon le déploiement AWS).

---

## Mapping des Champs Elasticsearch

```php
// Les champs Search API sont automatiquement mappés vers Elasticsearch
// Types de mapping :

// Search API type → Elasticsearch type
// text  → text (avec analyzer)
// string → keyword (pour les facettes/filtres exacts)
// integer → long
// decimal/float → double
// boolean → boolean
// date → date
// uri → keyword
```

---

## Vérifier la Connexion

```bash
# Test de connectivité
curl http://localhost:9200/

# Voir les index créés par Drupal
curl http://localhost:9200/_cat/indices?v

# Nombre de documents dans l'index
curl http://localhost:9200/drupal--articles_index--default--fr/_count

# Voir le mapping d'un index
curl http://localhost:9200/drupal--articles_index--default--fr/_mapping | python3 -m json.tool

# Recherche directe en Elasticsearch
curl -X GET http://localhost:9200/drupal--articles_index--default--fr/_search \
  -H "Content-Type: application/json" \
  -d '{"query": {"match": {"title": "drupal"}}}'
```

---

## Vider et Réindexer

```bash
# Depuis Drupal (recommandé)
drush sapi-c articles_index     # Vide l'index ES
drush sapi-i articles_index     # Réindexe

# Directement en Elasticsearch (si l'index est corrompu)
curl -X DELETE http://localhost:9200/drupal--articles_index--default--fr
drush sapi-i articles_index     # Recrée et réindexe

# Voir les logs d'indexation
docker compose logs elasticsearch | grep -E "error|ERROR" | tail -20
```

---

## Limitations vs Solr avec Drupal

```
❌ Pas de génération de schéma à déployer (search-api-solr:generate-solr-config n'a pas d'équivalent ES — le mapping est créé à la volée)
❌ Module moins testé en production Drupal
❌ Multilingue : pas d'analyseurs par langue aussi bien intégrés qu'avec Solr
❌ Highlighting : support variable selon le module utilisé
❌ Moins de documentation Drupal spécifique

✅ API REST native (pas de protocole propriétaire Solr)
✅ Kibana pour la visualisation des données
✅ Aggregations pour les analytics avancées
✅ Scalabilité horizontale native (si besoin)
```

**Pour la majorité des projets Drupal :** utiliser Solr. Pour des besoins d'analytics ou une équipe déjà sur la stack ELK : Elasticsearch.
