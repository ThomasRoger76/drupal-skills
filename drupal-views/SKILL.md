---
name: drupal-views
description: Use when configuring Drupal Views (lists, blocks, pages, RSS, REST exports), writing hook_views_data() to expose custom tables, creating custom field/filter/sort/relationship/area handlers, writing Views hook_views_query_alter() or hook_views_pre_render(), customizing Views Twig templates (views-view--VIEW-ID.html.twig, views-view-fields--VIEW-ID--DISPLAY-ID.html.twig), loading Views programmatically with Views::getView(), configuring exposed filters with AJAX or Better Exposed Filters, building contextual filters (arguments), setting up Views REST export for JSON output, optimizing Views cache, or debugging slow Views queries in Drupal 8-11+
---

# Drupal Views — Référence Complète

## Overview

Référentiel complet de Drupal Views 8-11+ : configuration UI, plugin system (handlers, display/style/row plugins), hooks d'altération, templates Twig, API PHP, REST export, cache, performance. Views est intégré au core Drupal depuis D8 (contrib D7).

## 🎯 La Règle Fondamentale

> **Views d'abord — code ensuite.** Si Views peut le faire via l'UI ou un handler, n'écris pas de requête SQL custom. Views gère automatiquement la langue, les permissions d'accès, le cache, la pagination, et les formats d'export.

---

## Quick Decision Table

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Liste de nœuds (page, bloc, flux RSS) | Views UI — Content display | [views-fundamentals.md](views-fundamentals.md) |
| Liste d'utilisateurs / termes de taxonomy | Views UI — type Users/Taxonomy | [views-fundamentals.md](views-fundamentals.md) |
| Filtres utilisateur sans rechargement | Exposed filters + AJAX | [views-ajax.md](views-ajax.md) |
| Filtres avancés (checkboxes, sliders, date range) | Module Better Exposed Filters | [views-ajax.md](views-ajax.md) |
| Filtrer selon un argument URL | Contextual filters (Arguments) | [views-fundamentals.md](views-fundamentals.md) |
| Exposer une table DB custom à Views | `hook_views_data()` | [hook-views-data.md](hook-views-data.md) |
| Modifier les données d'une table existante | `hook_views_data_alter()` | [hook-views-data.md](hook-views-data.md) |
| Afficher un champ calculé dans Views | Custom FieldHandler | [custom-handlers.md](custom-handlers.md) |
| Filtre custom sur une colonne spéciale | Custom FilterHandler | [custom-handlers.md](custom-handlers.md) |
| Tri custom (par formule, distance...) | Custom SortHandler | [custom-handlers.md](custom-handlers.md) |
| Relation custom entre deux tables | Custom RelationshipHandler | [custom-handlers.md](custom-handlers.md) |
| En-tête / pied de liste / texte si vide | Area handlers (Text, Entity, View) | [custom-handlers.md](custom-handlers.md) |
| Accès restreint à une View | Access plugin (permission, rôle, custom) | [views-fundamentals.md](views-fundamentals.md) |
| Charger une View depuis PHP | `Views::getView()` + `execute()` | [views-programmatic.md](views-programmatic.md) |
| Modifier la query SQL d'une View | `hook_views_query_alter()` | [views-programmatic.md](views-programmatic.md) |
| Modifier les résultats après exécution | `hook_views_post_execute()` | [views-programmatic.md](views-programmatic.md) |
| Injecter des données avant le rendu | `hook_views_pre_render()` | [views-programmatic.md](views-programmatic.md) |
| Template pour toute la View | `views-view--VIEW-ID.html.twig` | [views-templates.md](views-templates.md) |
| Template pour les champs d'une ligne | `views-view-fields--VIEW-ID--DISPLAY-ID.html.twig` | [views-templates.md](views-templates.md) |
| Template pour chaque ligne | `views-view-row-fields--VIEW-ID.html.twig` | [views-templates.md](views-templates.md) |
| Template pour une colonne spécifique | `views-view-field--FIELD-ID--VIEW-ID.html.twig` | [views-templates.md](views-templates.md) |
| Affichage en grille / liste personnalisée | Style plugin custom | [views-plugins.md](views-plugins.md) |
| Nouveau type de display (ex: iCal, Sitemap) | Display plugin custom | [views-plugins.md](views-plugins.md) |
| Exposer des données en JSON via Views | REST Export display | [views-rest-export.md](views-rest-export.md) |
| Authentification sur un REST export | `_format=json` + basic_auth / OAuth | [views-rest-export.md](views-rest-export.md) |
| Views + Search API (full-text search) | Backend Search API dans Views | [views-rest-export.md](views-rest-export.md) |
| Configurer le cache d'une View | Query settings → Caching | [views-cache.md](views-cache.md) |
| Cache tags custom sur une View | `hook_views_pre_render()` + addCacheTags | [views-cache.md](views-cache.md) |
| Débogage query Views | `hook_views_query_alter()` + `dpq($view->query)` | [views-programmatic.md](views-programmatic.md) |
| Générer une View programmatiquement | Agent `/drupal-generate-view` | [agents/views-generator.md](agents/views-generator.md) |
| Afficher les Paragraphs d'un nœud dans Views | Relationship + field handler SDC | [views-paragraphs.md](views-paragraphs.md) |
| View dont la source = paragraphes (pas nœuds) | Source : Paragraphs + relation nœud parent | [views-paragraphs.md](views-paragraphs.md) |
| Filtrer les nœuds par contenu d'un paragraphe | JOIN + filter sur paragraphs_item_field_data | [views-paragraphs.md](views-paragraphs.md) |
| **Actions en masse sur les résultats (bulk operations)** | Module `drupal/views_bulk_operations` — VBO actions configurables | [views-fundamentals.md](views-fundamentals.md) |
| **Export CSV/Excel de la View** | Module `drupal/views_data_export` — display "Data export" dans la View | [views-rest-export.md](views-rest-export.md) |
| **View avec infinite scroll / load more AJAX** | Module `drupal/views_infinite_scroll` — pager personnalisé | [views-ajax.md](views-ajax.md) |
| **View avec sticky header (tableau fixe)** | Module `drupal/datatables` ou CSS `position: sticky` sur le header Twig | [views-templates.md](views-templates.md) |
| **View avec relation Entity Reference multiple niveaux** | Relationship → Relationship → champ cible (3 niveaux max avant N+1) | [views-fundamentals.md](views-fundamentals.md) |
| **Exclure les doublons dans une View** | Query settings → "Distinct" + vérification de la pertinence avec DISTINCT SQL | [views-fundamentals.md](views-fundamentals.md) |
| **View avec tri par champ calculé / score** | Custom SortHandler + `hook_views_query_alter()` → `ORDER BY formule` | [custom-handlers.md](custom-handlers.md) |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| `db_query("SELECT...")` pour lister des entités | Créer une Vue Views | Pas de gestion langue/accès/cache |
| Template `views-fields--VIEW-ID.html.twig` | `views-view-fields--VIEW-ID--DISPLAY-ID.html.twig` | Template ignoré silencieusement |
| Views sans configuration de cache | Activer le cache (tag-based ou time-based) | Requête DB à chaque chargement |
| `Views::getView()` sans vérifier `!== NULL` | Toujours vérifier la vue existe + est activée | PHP Fatal si la vue est supprimée |
| Charger des entités dans `render()` du handler | Utiliser le batch loading (`loadMultiple`) | N+1 queries |
| Relation Views sans index DB | Ajouter un index sur la colonne de jointure | Query lente sur grandes tables |
| Access plugin "None" sur données sensibles | Access plugin "Permission" ou custom | Données exposées sans contrôle |
| Exposed filter sans AJAX sur liste longue | Activer AJAX sur le formulaire exposé | Rechargement complet de page |
| `$view->result` manipulé après `execute()` | Utiliser `hook_views_post_execute()` | Résultats incorrects ou non rendus |
| Contextual filter sans validation | Configurer "Provide default value" | 404 ou erreur si argument absent |

## Évolution par Version Majeure

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| Views intégré au core | ✅ | ✅ | ✅ | ✅ |
| Plugin annotations `@ViewsField` | ✅ | ✅ | ✅ | ⚠️ déprécié |
| PHP Attributes `#[ViewsField]` | ❌ | ❌ | ✅ optionnel | ✅ **standard** |
| jQuery dans Views AJAX | ✅ | ✅ | ❌ → vanilla JS | ❌ |
| `hook_views_data()` dans `.module` | ✅ | ✅ | ✅ | ✅ |
| REST Export (core) | ✅ | ✅ | ✅ | ✅ |
| Better Exposed Filters (contrib) | ✅ | ✅ | ✅ | ✅ |
| Search API backend | contrib | contrib | contrib | contrib |
| Views language join (D8.4+) | ✅ | ✅ | ✅ | ✅ |
| Tag-based caching Views | ✅ | ✅ | ✅ | ✅ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Bugs et pièges découverts en projet réel.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions.

## See Also

- `drupal-core` — EntityQuery, Entity API, plugin system
- `drupal-theming` — Templates Twig, template suggestions, preprocess
- `drupal-performance` — Cache tags, contexts, optimisation Views
- `drupal-config` — Export/import config Views (config/sync/views.view.*.yml)
- `drupal-search` — Search API + Views pour la recherche full-text
- `drupal-content-modeling` — Choisir entre Views et custom entity listing
