---
name: drupal-search — Views avec Search API
description: Construire des pages de recherche avec Views + Search API - configuration des champs de recherche, filtres exposés, tri par pertinence, intégration des facettes, et autocomplete.
---

# Views + Search API — Guide Complet

## Créer une View de Recherche

```
Views → Add new view
  → "Show" : Index des Articles (Search API Index)
  → "Create a page" : /recherche
  → Save and edit

Résultat : Views se connecte à Search API au lieu de la DB directement
```

---

## Champs dans une View Search API

```
Fields → Add

Différence avec une View normale :
  ├── Champs de l'index (définis dans search_api.index.*.yml)
  │     → "Indexed field: Titre", "Indexed field: Corps"
  │     → Rendu selon la configuration du champ dans l'index
  │
  ├── Champ "Excerpt" (Highlighting)
  │     → Affiche un extrait avec les termes surlignés
  │     → Requiert processor "Highlight" activé dans l'index
  │
  ├── Champ "Relevance score"
  │     → Score Solr/DB de pertinence — utile pour le tri
  │
  └── Champs de l'entité (via relationship)
        → Charger les champs depuis l'entité réelle (pas l'index)
        → Plus lent (charge les entités) mais plus flexible
```

---

## Filtre Fulltext — Configuration

```yaml
# Filter "Fulltext search" — permet de rechercher dans les champs indexés
filters:
  search_api_fulltext:
    id: search_api_fulltext
    table: search_api_index_articles_index
    field: search_api_fulltext
    plugin_id: search_api_fulltext
    exposed: true
    expose:
      label: 'Rechercher'
      identifier: keys
      placeholder: 'Saisir vos mots-clés...'
    value: ''
    fields:                        # Champs dans lesquels rechercher
      title: title
      body: body
    min_length: 3                  # Longueur minimale de recherche
```

**Via l'UI :**
```
Filters → Add → Chercher "Search: Fulltext search"
  → Expose to visitors : ✅
  → Identifier : keys (apparaît dans l'URL : ?keys=drupal)
  → Configure fields to search : titre, corps, résumé
```

---

## Tri par Pertinence

```yaml
# Sort criteria : trier par score Solr
sort_criteria:
  search_api_relevance:
    id: search_api_relevance
    table: search_api_index_articles_index
    field: search_api_relevance
    order: DESC                    # DESC = du plus pertinent au moins pertinent
    exposed: false                 # Pas exposé — toujours pertinence en premier
```

**Tri exposé (utilisateur peut choisir) :**
```yaml
sort_criteria:
  search_api_relevance:
    id: search_api_relevance
    exposed: true
    expose:
      label: 'Pertinence'
      field_identifier: relevance
  created:
    id: created
    table: search_api_index_articles_index
    field: created
    exposed: true
    expose:
      label: 'Date'
      field_identifier: date
```

---

## Intégrer les Facettes dans la View

```bash
# 1. Installer Facets (si pas encore fait)
composer require drupal/facets
drush en facets -y

# 2. Créer une facette pour la View
# /admin/config/search/facets/add
# → Field : field_tags (indexed)
# → Source : search_api:views_page__recherche__page_1
# → Widget : Checkboxes

# 3. Placer le bloc de facette dans une région
# /admin/structure/block
# → Trouver "Facet: Tags" → placer en sidebar_first
# → Limiter à la page /recherche
```

---

## Afficher les Résultats de Recherche

```twig
{# Template Views adapté à la recherche #}
{# templates/views/views-view-fields--recherche--page-1.html.twig #}

<article class="search-result">
  <h2 class="search-result__titre">
    <a href="{{ fields.url.content }}">{{ fields.title.content }}</a>
  </h2>

  {% if fields.search_api_excerpt.content %}
    {# L'excerpt contient les highlights Solr - ne pas échapper #}
    <p class="search-result__excerpt">
      {{- fields.search_api_excerpt.content|raw -}}
    </p>
  {% else %}
    <p class="search-result__body">{{ fields.body.content }}</p>
  {% endif %}

  <div class="search-result__meta">
    <time>{{ fields.created.content }}</time>
    {% if fields.field_tags_name.content %}
      <span class="search-result__tags">{{ fields.field_tags_name.content }}</span>
    {% endif %}
  </div>
</article>
```

---

## Autocomplete avec search_api_autocomplete

```bash
composer require drupal/search_api_autocomplete
drush en search_api_autocomplete -y
```

```yaml
# config/install/search_api_autocomplete.search.articles_search.yml
langcode: fr
status: true
id: articles_search
label: 'Autocomplete Articles'
index_id: articles_index
suggester:
  id: server
  configuration:
    fields:
      title: title
    suggesters:
      - server
overwrite_style: true
options:
  limit: 5
  min_length: 2
  show_count: false
  delay: 200
```

**Intégration dans la View :**
```
Views → Filter "Fulltext search" → More
→ Enable autocomplete : ✅
→ Choose a search component : "Articles Search"
```

---

## Résumé de la Configuration de la Recherche

```
Page de recherche complète =
  ├── View "recherche"
  │     ├── Source : Index Search API
  │     ├── Filter exposé : Fulltext search (identifier: keys)
  │     ├── Filter exposé : type (optionnel)
  │     ├── Sort : Relevance DESC (défaut), Date DESC (optionnel exposé)
  │     ├── Fields : titre, excerpt, date, tags
  │     ├── Pager : Full, 20 items/page
  │     ├── AJAX : YES
  │     └── Cache : Tag-based
  │
  ├── Facettes (blocs en sidebar)
  │     ├── Facette tags → checkboxes
  │     ├── Facette type → radio buttons
  │     └── Facette date → range
  │
  └── Autocomplete
        → Sur le champ de recherche, suggestions en temps réel
```

---

## Debug et Performance

```bash
# Vérifier les termes recherchés et les résultats
drush php:eval "
\$query = \Drupal::entityTypeManager()
  ->getStorage('search_api_index')
  ->load('articles_index')
  ->query();
\$query->keys('drupal');
\$query->addCondition('status', 1);
\$query->range(0, 10);
\$results = \$query->execute();
echo 'Total: ' . \$results->getResultCount() . PHP_EOL;
foreach (\$results->getResultItems() as \$id => \$item) {
  echo \$id . PHP_EOL;
}
"

# Voir la query Solr envoyée (enable query logging)
# /admin/config/search/search-api/server/solr_server/view
# → "Test server" → saisir une requête → voir la query Solr

# Performance : pas de relations dans les Views Search API
# → charger les entités uniquement si nécessaire
# → utiliser les champs indexés au maximum
```
