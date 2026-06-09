---
name: drupal-search — solr integration
description: Intégration Solr avec Search API Drupal - configuration du serveur, génération du schéma, field mapping, boost de relevance, highlighting, et multilingue.
---

# Search API Solr — Référence Complète

## Installation

```bash
# Module Drupal
composer require drupal/search_api_solr
drush en search_api_solr -y

# Service Solr dans docker-compose.yml
solr:
  image: solr:9-slim
  ports:
    - "8983:8983"
  volumes:
    - solr_data:/var/solr
    - ./services/solr/drupal:/opt/solr/server/solr/drupal:ro
  command: solr-precreate drupal /opt/solr/server/solr/drupal
  environment:
    SOLR_JAVA_MEM: "-Xms512m -Xmx512m"

volumes:
  solr_data:
```

---

## Générer et Déployer le Schéma Solr

```bash
# 1. Configurer le serveur Solr dans Drupal
#    /admin/config/search/search-api/server/add

# 2. Configurer l'index et ses champs

# 3. Générer le schéma Solr depuis Drupal
# Commande complète (toutes versions du module) :
drush search-api-solr:generate-solr-config mon_index /tmp/solr-config/

# Alias court (même commande) :
drush sapi-solr:gsc mon_index /tmp/solr-config/
# NOTE : 'drush solr-gsc' est un alias OBSOLÈTE de versions antérieures du module
# Toujours utiliser search-api-solr:generate-solr-config

# 4. Copier les fichiers de config dans le core Solr
docker compose cp /tmp/solr-config/. solr:/opt/solr/server/solr/drupal/conf/

# 5. Recharger le core Solr (quoter l'URL — sinon le shell interprète & comme un job)
curl "http://localhost:8983/solr/admin/cores?action=RELOAD&core=drupal"

# 6. Vérifier le core
curl "http://localhost:8983/solr/drupal/select?q=*:*&rows=0"
```

### Structure des fichiers de config Solr générés

```
/tmp/solr-config/
├── schema.xml                    ← Définition des champs Solr
├── solrconfig.xml               ← Configuration du core
├── managed-schema               ← Schéma alternatif (Solr 6+)
├── elevate.xml                  ← Promotion manuelle de résultats
└── stopwords.txt, synonyms.txt  ← Traitement du texte
```

---

## Configuration du Serveur via YAML

```yaml
# config/install/search_api.server.solr_server.yml
langcode: fr
status: true
id: solr_server
name: 'Solr Server'
description: ''
backend: search_api_solr
backend_config:
  connector: standard
  connector_config:
    scheme: http
    host: solr              # Nom du service Docker
    port: '8983'
    path: /
    core: drupal
    timeout: 5
    index_timeout: 5
    query_timeout: 5
    finalize_timeout: 30
  retrieve_data: false
  highlight_data: true       # Activer le highlighting
  skip_schema_check: false
  solr_version: '9'
  commit_within: 1000        # Commit Solr dans les 1000ms
  jmx_enabled: false
```

---

## Field Mapping et Boost de Relevance

```yaml
# Dans la configuration de l'index Search API :
field_settings:
  title:
    label: Titre
    datasource_id: 'entity:node'
    property_path: title
    type: text
    boost: '21.0'           # ← Boost élevé = titre très important

  body:
    label: Corps
    datasource_id: 'entity:node'
    property_path: body
    type: text
    boost: '1.0'            # ← Boost normal

  field_tags_name:
    label: 'Nom des tags'
    datasource_id: 'entity:node'
    property_path: 'field_tags:entity:name'
    type: string
    boost: '3.0'            # ← Tags moyennement importants

  field_summary:
    label: 'Résumé'
    datasource_id: 'entity:node'
    property_path: 'body:summary'
    type: text
    boost: '5.0'            # ← Résumé plus important que le corps
```

---

## Highlighting — Surligner les Termes Recherchés

```yaml
# Activer le highlighting dans la config du serveur
backend_config:
  highlight_data: true

# Activer le processeur "Highlight" dans l'index
processor_settings:
  highlight:
    processor_id: highlight
    weights:
      postprocess_query: 0
      preprocess_index: 0
    settings:
      prefix: '<strong>'   # Balise ouvrante pour le highlight
      suffix: '</strong>'  # Balise fermante
      excerpt: true        # Générer un extrait avec contexte
      excerpt_length: 256  # Longueur de l'extrait
      exclude_fields: []   # Champs à ne pas surligner
      highlight: always    # 'always', 'server', 'never'
```

```twig
{# Afficher le résultat avec highlighting dans Twig #}
{# Views → Champs → Utiliser "Excerpt" depuis Search API #}

<article class="search-result">
  <h2><a href="{{ url }}">{{ title }}</a></h2>
  {# L'excerpt contient déjà les balises <strong> du highlighting #}
  <p class="search-result__excerpt">{{ excerpt|raw }}</p>
</article>
```

---

## Recherche Multilingue

### Analyseurs par Langue (Solr)

```xml
<!-- schema.xml — analyseur spécifique par langue -->
<!-- Drupal Search API Solr génère automatiquement ces configs -->

<fieldType name="text_fr" class="solr.TextField" positionIncrementGap="100">
  <analyzer type="index">
    <tokenizer class="solr.StandardTokenizerFactory"/>
    <filter class="solr.LowerCaseFilterFactory"/>
    <filter class="solr.ElisionFilterFactory" articles="lang/contractions_fr.txt"/>
    <filter class="solr.StopFilterFactory" words="lang/stopwords_fr.txt" ignoreCase="true"/>
    <filter class="solr.FrenchLightStemFilterFactory"/>
  </analyzer>
  <analyzer type="query">
    <tokenizer class="solr.StandardTokenizerFactory"/>
    <filter class="solr.LowerCaseFilterFactory"/>
    <filter class="solr.ElisionFilterFactory" articles="lang/contractions_fr.txt"/>
    <filter class="solr.StopFilterFactory" words="lang/stopwords_fr.txt" ignoreCase="true"/>
    <filter class="solr.FrenchLightStemFilterFactory"/>
  </analyzer>
</fieldType>
```

### Configuration Multilingue Search API

```yaml
# Ajouter le processeur Language with Fallback
processor_settings:
  language_with_fallback:
    processor_id: language_with_fallback
    weights:
      preprocess_index: -20
      preprocess_query: -20
    settings: {}
  # Permet d'afficher le contenu en langue d'origine si la traduction manque
```

---

## Debugger les Requêtes Solr

```bash
# Activer le debug Solr (montre la query Solr dans Drupal)
# /admin/config/search/search-api → View server → Test server

# Voir la requête Solr directement
drush php:eval "
\$index = \Drupal::entityTypeManager()->getStorage('search_api_index')->load('articles_index');
\$query = \$index->query(['limit' => 5]);
\$query->keys('drupal');
\$result = \$query->execute();
echo 'Total: ' . \$result->getResultCount() . PHP_EOL;
foreach (\$result->getResultItems() as \$item) {
  echo \$item->getId() . ': ' . \$item->getField('title')->getValues()[0] . PHP_EOL;
}
"

# Interface Solr admin — voir les queries
# http://localhost:8983/solr/#/drupal/query

# Test direct Solr
curl "http://localhost:8983/solr/drupal/select?q=drupal&fl=id,title_fr&rows=5&indent=true"

# Vérifier le schéma
curl http://localhost:8983/solr/drupal/schema/fields | python3 -m json.tool | head -50

# Compter les documents dans Solr
curl "http://localhost:8983/solr/drupal/select?q=*:*&rows=0" | python3 -m json.tool
```

---

## Monitoring et Maintenance Solr

```bash
# Stats du core Solr
curl http://localhost:8983/solr/drupal/admin/luke?numTerms=0

# Optimiser le core (défragmenter)
curl "http://localhost:8983/solr/drupal/update?optimize=true&waitFlush=false"

# Vider le core (tout supprimer — puis réindexer)
curl -X POST "http://localhost:8983/solr/drupal/update" \
  -H "Content-Type: text/xml" \
  -d '<delete><query>*:*</query></delete>'
curl -X POST "http://localhost:8983/solr/drupal/update?commit=true"

# Puis depuis Drupal
drush sapi-i articles_index
```
