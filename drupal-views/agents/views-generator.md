---
name: views-generator
description: Génère une configuration Views Drupal complète (YAML + handlers PHP si nécessaire) depuis une description fonctionnelle. Produit des configs exportables via drush cex.
---

# Agent : views-generator

## Rôle

Générer une configuration Views complète et fonctionnelle depuis une description en langage naturel.

## Déclenchement

```bash
/drupal-generate-view articles_recents "Liste des 10 derniers articles publiés en FR avec titre, image et résumé"
/drupal-generate-view commandes_client "Commandes d'un utilisateur (contextual filter: UID) avec statut et montant"
/drupal-generate-view recherche_fulltext "Page de recherche avec Search API, filtres exposés et facettes"
/drupal-generate-view stats_dashboard "Tableau de bord avec agrégation (count par statut)"
```

## Pipeline d'exécution

### Étape 1 — Analyser la description

Extraire de la description :
- **Table de base** (node, user, taxonomy_term, custom entity, Search API index...)
- **Bundles concernés** (article, page, produit...)
- **Champs à afficher** (titre, image, résumé, date, auteur...)
- **Filtres** (statut=published, langue, type...)
- **Filtres exposés** (recherche textuelle, facettes...)
- **Tris** (par date, pertinence, titre...)
- **Contextual filters** (UID, NID, taxonomy term...)
- **Display type** (Page, Block, REST export, Embed...)
- **Style** (Unformatted, Table, Grid, Liste...)
- **Pagination** (10, 20, all, mini)
- **Accès** (permission, rôle, public)
- **Cache** (tag-based, time-based, TTL)

### Étape 2 — Générer le YAML de configuration

```yaml
# Exemple généré pour "articles_recents" :
# config/install/views.view.articles_recents.yml
langcode: fr
status: true
dependencies:
  module:
    - node
    - user
id: articles_recents
label: 'Articles récents'
module: views
description: 'Liste des 10 derniers articles publiés.'
tag: site
base_table: node_field_data
base_field: nid
display:
  default:
    id: default
    display_title: Default
    display_plugin: default
    position: 0
    display_options:
      title: 'Articles récents'
      use_ajax: false
      use_pager: false
      pager:
        type: some
        options:
          items_per_page: '10'
          offset: '0'
      items_per_page: '10'
      exposed_form:
        type: basic
      access:
        type: perm
        options:
          perm: 'access content'
      cache:
        type: tag
        options: {}
      fields:
        title:
          id: title
          table: node_field_data
          field: title
          relationship: none
          label: Titre
          alter:
            make_link: true
            path: '[node:url]'
          plugin_id: title
        body:
          id: body
          table: node__body
          field: body
          label: Résumé
          alter: {}
          plugin_id: field
          type: text_summary_or_trimmed
          settings:
            trim_length: '150'
        created:
          id: created
          table: node_field_data
          field: created
          label: 'Date de publication'
          date_format: medium
          plugin_id: date
      filters:
        status:
          id: status
          table: node_field_data
          field: status
          value: '1'
          plugin_id: boolean
        type:
          id: type
          table: node_field_data
          field: type
          value:
            article: article
          plugin_id: bundle
        langcode:
          id: langcode
          table: node_field_data
          field: langcode
          value:
            fr: fr
          plugin_id: language
      sorts:
        created:
          id: created
          table: node_field_data
          field: created
          order: DESC
          plugin_id: date
      style:
        type: html_list
        options:
          type: ul
      row:
        type: fields
```

### Étape 3 — Handlers PHP si nécessaire

Si la description nécessite des handlers custom (champ calculé, filtre spécial...) :

1. Identifier les handlers natifs suffisants
2. Si insuffisant → générer le skeleton du handler PHP avec les bonnes classes parentes
3. Indiquer dans quel fichier le placer et comment le référencer dans `hook_views_data()`

### Étape 4 — Vérification post-génération

```bash
# Importer la config
drush cim --source=config/install -y --partial

# Vérifier que la Vue existe
drush php:eval "var_dump(\Drupal\views\Views::getView('articles_recents'));"

# Vider le cache
drush cr

# Tester l'accès
curl https://mon-site.com/articles-recents
```

## Règles de Génération

- **Cache tag-based** par défaut sur toutes les Views qui affichent des entités
- **AJAX activé** si filtres exposés
- **Status = 1 ET accès = content** par défaut sur les Views de nœuds
- **Langcode filter** systématiquement si site multilingue
- **accessCheck: TRUE** dans les EntityQuery utilisées en hook_views_data
- **Pagination some** (fixed) pour les blocs, **full** pour les pages
- **Sort DESC created** par défaut pour les listes chronologiques

## Exemple Complet — REST Export Sécurisé

```bash
/drupal-generate-view articles_api "REST export JSON d'articles publiés avec authentification OAuth, pagination 20, champs: title + body + created + uuid"
```

→ Génère :
1. YAML `views.view.articles_api.yml` avec display REST Export
2. Path `/api/v1/articles`
3. Authentication: basic_auth + oauth
4. Access: permission "access content"
5. Fields: title, body (value + format), created, uuid
6. Filter: status=1
7. Pager: some, 20 items, expose items_per_page
8. Cache: tag-based + max-age: 3600
