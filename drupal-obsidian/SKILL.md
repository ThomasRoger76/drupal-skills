---
name: drupal-obsidian
description: "Use when mapping a Drupal 10/11 project architecture into an Obsidian vault for living documentation: extracting Content Types, Paragraphs, Fields, Views, Migrations, Workflows, and SDC Components from config/sync YAML into linked Markdown notes with Dataview queries for architecture audits, dependency graphs, and coverage reports. Includes Python extraction script, vault structure, wikilinks mapping, and graphify integration for interactive knowledge graphs."
---

# Drupal → Obsidian Architecture Mapping

## Overview

Ce skill transforme la configuration Drupal (`config/sync/*.yml`) en un **vault Obsidian vivant** : chaque Content Type, Paragraph, champ, template Twig et hook devient une note liée. Le résultat est une cartographie de l'architecture explorable en Graph View et requêtable avec Dataview — puis transformable en graphe interactif via le skill `graphify`.

## Pourquoi un Vault Drupal ?

Un projet Drupal de taille moyenne contient 50-200 entités (Content Types, Paragraphs, Views, Workflows, Migrations) réparties dans des centaines de fichiers YAML dans `config/sync/`. La documentation manuelle devient obsolète dès le sprint suivant. Ce skill résout ce problème en **générant automatiquement une documentation vivante** depuis la source de vérité : `config/sync/*.yml`.

**Ce que le vault permet qu'un wiki manuel ne peut pas :**
- Graph View : voir instantanément quels modules dépendent de quel champ
- Dataview : auditer les paragraphes sans template Twig en une requête
- Auto-update en CI/CD : la doc est toujours synchronisée avec le code
- `git diff` sur les notes : savoir ce qui a changé entre deux sprints

## La Chaîne Complète

```
config/sync/*.yml
      ↓ (scripts/extraction.py ou drush)
Obsidian Vault (notes .md avec frontmatter + [[wikilinks]])
      ↓ (Graph View + Dataview)
Architecture vivante + Audit automatisé
      ↓ (skill graphify)
HTML Knowledge Graph interactif + Rapport d'audit
```

## Quick Decision Table

### Extraction — Script Python

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Extraire les Content Types → notes Obsidian | Script Python → `type: content_type` | [extraction.md](extraction.md) |
| Extraire les Paragraphs → notes | Script Python → `type: paragraph` | [extraction.md](extraction.md) |
| Extraire les Fields (storage + instances) → notes | Script Python → `type: field_storage/field_instance` | [extraction.md](extraction.md) |
| Extraire les Views → notes | Script Python → `type: view` | [extraction.md](extraction.md) |
| Extraire les Workflows (Content Moderation) → notes | Script Python → `type: workflow` | [extraction.md](extraction.md) |
| Extraire les Migrations → notes | Script Python → `type: migration` | [extraction.md](extraction.md) |
| Extraire les SDC Components → notes | Script Python → `type: sdc_component` | [extraction.md](extraction.md) |
| Extraire les Drupal Recipes → notes | Script Python → `type: recipe` | [extraction.md](extraction.md) |
| Extraire les Menus et liens de menu | Script Python → `type: menu` | [extraction.md](extraction.md) |
| Extraire les Rôles et permissions | Script Python → `type: role` | [extraction.md](extraction.md) |
| Extraire les Media Types | Script Python → `type: media_type` | [extraction.md](extraction.md) |
| Extraire les Entity View Displays | Script Python → `type: view_display` | [extraction.md](extraction.md) |
| Extraire les Taxonomies | Script Python → `type: taxonomy` | [extraction.md](extraction.md) |
| Détecter automatiquement le type d'entité depuis le nom de fichier | `detect_entity_type(filename, content)` | [extraction.md](extraction.md) |
| Gérer les YAML malformés sans crasher | `safe_load_yaml()` avec try/except | [extraction.md](extraction.md) |
| Extraction parallèle sur 1000+ fichiers YAML | `ThreadPoolExecutor` + `extract_all_parallel()` | [extraction.md](extraction.md) |
| Extraire les champs d'un Content Type depuis config/sync | `extract_fields_from_yaml()` | [extraction.md](extraction.md) |

### Structure du Vault

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Organiser les dossiers du vault | Architecture standard vault | [vault-structure.md](vault-structure.md) |
| Workflow Git pour vault en équipe | `.gitignore` + merge conflicts | [vault-structure.md](vault-structure.md) |
| Mettre à jour le vault en CI/CD | Script Python + `.gitlab-ci.yml` | [vault-structure.md](vault-structure.md) |
| Comparer architecture avant/après sprint | `git diff` sur les notes | [vault-structure.md](vault-structure.md) |
| Intégrer les notes Obsidian dans GitLab Wiki | Script Python → export Markdown GitLab | [vault-structure.md](vault-structure.md) |
| **CI/CD — regénérer le vault automatiquement** | `.gitlab-ci.yml` → `python3 scripts/extraction.py --config-dir config/sync --output-dir vault/` | [vault-structure.md](vault-structure.md) |
| **Workflow équipe — vault partagé via Git** | Vault dans un sous-dossier git du projet Drupal + `.gitignore` pour `/.obsidian/workspace.json` | [vault-structure.md](vault-structure.md) |
| **Comparer l'architecture entre deux branches** | `git diff main feature/refacto -- vault/` → noter les entités ajoutées/supprimées | [vault-structure.md](vault-structure.md) |
| **Dashboard sprint — contenu non traduit** | Dataview → `type = "content_type" AND NOT file.inlinks.some(...)` | [dataview-queries.md](dataview-queries.md) |
| **Audit des champs sans formatters définis** | Dataview JS → `type: "field_instance"` sans `#formatter` tag | [dataview-queries.md](dataview-queries.md) |
| **Rapport des migrations orphelines** | Dataview → `type: "migration"` sans lien vers un Content Type destination | [dataview-queries.md](dataview-queries.md) |

### Templates de Notes

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Template note Content Type | Frontmatter + champs + wikilinks | [note-templates.md](note-templates.md) |
| Template note Paragraph | Frontmatter + champs + bundles parents | [note-templates.md](note-templates.md) |
| Template note Migration | Source / Destination / Dépendances | [note-templates.md](note-templates.md) |
| Template note Workflow | États, transitions, permissions | [note-templates.md](note-templates.md) |
| Template note SDC Component | Props, slots, exemple Twig | [note-templates.md](note-templates.md) |
| Template note Recipe | Config actions, modules installés | [note-templates.md](note-templates.md) |
| Template note champ réutilisable | Frontmatter + backlinks | [note-templates.md](note-templates.md) |
| Template note template Twig | Frontmatter + mapping config | [note-templates.md](note-templates.md) |

### Wikilinks et Cartographie

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Lier entités ↔ champs ↔ templates | Wikilinks bidirectionnels | [wikilinks-mapping.md](wikilinks-mapping.md) |
| Mapper les routes vers les Content Types | `type: route` + wikilinks | [wikilinks-mapping.md](wikilinks-mapping.md) |
| Documenter les services custom | `type: service` dans le vault | [wikilinks-mapping.md](wikilinks-mapping.md) |
| Mapper les hooks d'un module | `type: hook` + outlinks vers CT | [wikilinks-mapping.md](wikilinks-mapping.md) |

### Dataview — Audit et Requêtes

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Paragraphs sans template Twig | Dataview JS → backlinks manquants | [dataview-queries.md](dataview-queries.md) |
| Champs critiques (utilisés partout) | Dataview → backlinks count | [dataview-queries.md](dataview-queries.md) |
| Modules sans tests | Dataview → modules sans note `#test` | [dataview-queries.md](dataview-queries.md) |
| Templates orphelins | Dataview → `#twig_template` sans outlink CT | [dataview-queries.md](dataview-queries.md) |
| Migrations non documentées | Dataview → `#migration` sans description | [dataview-queries.md](dataview-queries.md) |
| Coverage tests par module | Dataview → comptage `#test` par module | [dataview-queries.md](dataview-queries.md) |
| Dashboard santé du vault | Dataview → comptages globaux par type | [dataview-queries.md](dataview-queries.md) |
| Entités sans access control documenté | Dataview → `#entity` sans `access_handler` | [dataview-queries.md](dataview-queries.md) |
| Views sans filtre exposé documenté | Dataview → `#view` sans `exposed_filters` | [dataview-queries.md](dataview-queries.md) |
| Tableau Entity Reference → Taxonomies | Dataview query | [dataview-queries.md](dataview-queries.md) |
| Exporter les résultats Dataview en CSV | DataviewJS → export CSV | [dataview-queries.md](dataview-queries.md) |
| Valider la cohérence des noms BEM | Dataview → cross-ref classes CSS | [dataview-queries.md](dataview-queries.md) |

### Graphify et Visualisation

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Générer un knowledge graph interactif | skill `graphify` | [graphify-integration.md](graphify-integration.md) |
| Visualiser le flux de données Config → Twig | Canvas Obsidian | [graphify-integration.md](graphify-integration.md) |
| Identifier les dépendances critiques | Graphify + filtres par type | [graphify-integration.md](graphify-integration.md) |

## Typage des Notes — Frontmatter Standard

Chaque note du vault utilise `type:` en frontmatter pour que Dataview puisse filtrer :

```
type: content_type   → Content Types (node.type.*)
type: paragraph      → Paragraphs (paragraphs.paragraphs_type.*)
type: taxonomy       → Vocabulaires (taxonomy.vocabulary.*)
type: media          → Types Media
type: field_storage  → Stockages de champs (field.storage.*)
type: field_instance → Instances de champs (field.field.*)
type: view_display   → Modes d'affichage (core.entity_view_display.*)
type: twig_template  → Templates .html.twig
type: view           → Views (views.view.*)
type: hook           → Hooks PHP
type: service        → Services Drupal
type: workflow       → Workflows éditoriaux
```

## Anti-Patterns

| ❌ À ne pas faire | ✅ Bonne pratique | Raison |
|------------------|------------------|--------|
| Tout extraire en une note géante | Une note par entité | Backlinks et Graph View inutilisables |
| Wikilinks vers des notes inexistantes | Créer la note cible ou utiliser un tag | Graph View affiche des orphelins trompeurs |
| Frontmatter incohérent entre notes | Utiliser les templates de note-templates.md | Dataview queries échouent silencieusement |
| Régénérer le vault manuellement | Script d'extraction automatisé | Désynchronisation avec la config réelle |
| Graphify sur l'ensemble du vault sans filtre | Pré-filtrer par dossier/tag | Graphe illisible avec 200+ nœuds |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Problèmes rencontrés lors du mapping
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions (v1.0 courante)

## See Also

- `drupal-config` — Config Management System (source des YAML)
- `drupal-core` — Entity API, Config Entities (ce qu'on mappe)
- `drupal-theming` — Templates Twig (destination du mapping)
- `graphify` — Knowledge graph interactif (visualisation finale)
- `obsidian-markdown` — Syntax Obsidian, wikilinks, callouts
