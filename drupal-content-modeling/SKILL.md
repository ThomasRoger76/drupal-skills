---
name: drupal-content-modeling
description: Use when deciding between Drupal content architecture approaches (Nodes vs Paragraphs vs Layout Builder vs Custom Entities vs Taxonomy), designing Paragraphs types with nested paragraphs or behaviors (ParagraphsBehaviorInterface), choosing the right field type (String vs Text vs Body, Date vs DateTime vs Daterange, File vs Image vs Media), setting up content types with the right field configuration, implementing custom Content Entities with bundles and access control, using Taxonomy for classification vs navigation vs free tagging, avoiding N+1 queries with Paragraphs, migrating from Paragraphs to Layout Builder, modeling multi-tenant content with Group module, or deciding on content architecture for headless Drupal in Drupal 8-11+
---

# Drupal Content Modeling — Architecture & Référence Complète

## Overview

Guide de référence pour les décisions d'architecture de contenu Drupal 8-11+ : quand utiliser quoi entre Nodes, Paragraphs, Layout Builder, Custom Entities, Taxonomy, et Block. Inclut la sélection des types de champs, la modélisation des relations, et les implications en termes de performance, migration, et maintenabilité.

## 🎯 La Règle Fondamentale

> **La structure suit le besoin éditorial.** L'architecture de contenu Drupal doit être choisie selon les besoins des éditeurs et des performances — pas selon les habitudes du développeur. Une mauvaise décision au départ est coûteuse à défaire.

---

## Quick Decision Table — Choisir la Structure

| Besoin | Solution | Pourquoi | Référence |
|--------|----------|----------|-----------|
| Contenu SEO avec URL propre (article, page) | **Node** (Content Type) | Routage, alias, métadonnées natifs | [nodes-content-types.md](nodes-content-types.md) |
| Mise en page variable par composants empilés | **Paragraphs** | Flexibilité éditoriale, composants réutilisables | [paragraphs.md](paragraphs.md) |
| Mise en page personnalisée page par page | **Layout Builder** | L'éditeur place les blocs visuellement | [layout-builder.md](layout-builder.md) |
| Données métier sans frontend (réservations, commandes) | **Custom Entity** | Pas les surcharges d'un node (révisions, path...) | [custom-entities.md](custom-entities.md) |
| Classification de contenu (genre, thématique) | **Taxonomy** | Native, Views-compatible, URL aliases | [taxonomy-advanced.md](taxonomy-advanced.md) |
| Tags libres entrés par les utilisateurs | **Taxonomy vocabulaire free tagging** | Auto-complete, création à la volée | [taxonomy-advanced.md](taxonomy-advanced.md) |
| Navigation principale / secondaire | **Menu links** (pas Taxonomy) | API Menu native, traduction, poids | [nodes-content-types.md](nodes-content-types.md) |
| Relation Many-to-Many entre entités | **Entity Reference** (champ) | Joins DB, Views avec relation | [field-types.md](field-types.md) |
| Données transactionnelles haute fréquence | **Custom table** + `hook_schema` | Pas de surcharge Entity API | [custom-entities.md](custom-entities.md) |
| Composants avec configuration dev (pas éditeur) | **Block custom** (plugin PHP) | Configurable mais pas éditable en UI | [nodes-content-types.md](nodes-content-types.md) |
| Bloc éditorial avec champs (CTA, citation, bannière) | **Block Content** (`block_content`) | Entité avec champs, réutilisable, traductible | [block-content-types.md](block-content-types.md) |
| Bloc existant réutilisé dans Layout Builder | **Block Content** → Reuse existing block | Même entité dans plusieurs régions/pages | [block-content-types.md](block-content-types.md) |
| Paragraphs vs Layout Builder — lequel choisir ? | Voir tableau comparatif | Critères fonctionnels et techniques | [paragraphs.md](paragraphs.md) |
| Paragraphes imbriqués (sections > colonnes > blocs) | **Nested Paragraphs** — avec précaution | Max 3 niveaux, impact perfs | [paragraphs.md](paragraphs.md) |
| Comportement conditionnel selon le type de para | **Paragraphs Behaviors** | Plugin `ParagraphsBehaviorInterface` | [paragraphs.md](paragraphs.md) |
| Champ texte court (titre alternatif, sous-titre) | **String** field | Pas de format texte, performant | [field-types.md](field-types.md) |
| Champ texte multi-lignes sans HTML | **Text (plain long)** | Textarea sans WYSIWYG | [field-types.md](field-types.md) |
| Champ WYSIWYG (corps d'article) | **Text with summary** (Body) | Format texte configurable, résumé | [field-types.md](field-types.md) |
| Date seule (anniversaire, date de publication) | **Date** (YYYY-MM-DD) | Pas de timezone | [field-types.md](field-types.md) |
| Date + heure (événement précis) | **DateTime** (YYYY-MM-DDTHH:MM:SS) | Avec timezone | [field-types.md](field-types.md) |
| Plage de dates (événement avec début/fin) | **Daterange** | Deux dates liées, filtrable en Views | [field-types.md](field-types.md) |
| Image avec alt, title, crop | **Image** field + focal_point | Alt obligatoire WCAG, crop responsive | [field-types.md](field-types.md) |
| Vidéo, document, image dans Media Library | **Media Entity Reference** | Réutilisable, Media Library native | [field-types.md](field-types.md) |
| Lien externe ou interne | **Link** field | Validation URL, texte du lien | [field-types.md](field-types.md) |
| Champ calculé (non stocké) | **Computed field** via `#[FieldType]` | Valeur calculée à la lecture | [custom-entities.md](custom-entities.md) |
| Entity avec plusieurs bundles (comme node types) | **ContentEntityBase** + bundle entity | Pattern Drupal natif | [custom-entities.md](custom-entities.md) |
| Accès granulaire par entité (multi-tenant) | **AccessControlHandler** custom | + éventuellement Group module | [custom-entities.md](custom-entities.md) |
| **Page builder drag-and-drop pour les éditeurs** | `drupal/dxpr_builder` — alternative visuelle à Layout Builder | [layout-builder.md](layout-builder.md) |
| **Paragraphs en sections avec colonnes** | `drupal/layout_paragraphs` — Layout Builder UI dans les Paragraphs | [paragraphs.md](paragraphs.md) |
| **Transitions de statut planifiées** | `drupal/scheduled_transitions` — publier/dépublier à une date | [nodes-content-types.md](nodes-content-types.md) |
| Verrouiller un contenu en cours d'édition | `drupal/content_lock` — empêche l'édition simultanée | [nodes-content-types.md](nodes-content-types.md) |
| Cloner un nœud / une entité | `drupal/entity_clone` — duplication via UI ou programmatique | [nodes-content-types.md](nodes-content-types.md) |
| **Workflow complexe multi-étapes (ECA)** | `drupal/eca` — Event-Condition-Action sans code, remplace Rules en D10/D11 | [nodes-content-types.md](nodes-content-types.md) |
| **Archiver du contenu après X jours automatiquement** | `drupal/eca` + Timer event → transition vers état `archived` | [nodes-content-types.md](nodes-content-types.md) |
| **Indexer un contenu custom dans Search API** | `@SearchApiDatasource` plugin → expose l'entité custom au moteur de recherche | [custom-entities.md](custom-entities.md) |
| **Révisions avec diff visuel (avant/après)** | `drupal/diff` — affiche les changements entre deux révisions | [nodes-content-types.md](nodes-content-types.md) |
| **Versionner le contenu comme du code (git-like)** | Content Moderation + Revisions + `drupal/workspaces` (staging en DB) | [nodes-content-types.md](nodes-content-types.md) |
| **Multi-step content approval workflow** | Content Moderation → états custom (draft → in_review → approved → published) | [nodes-content-types.md](nodes-content-types.md) |

## Paragraphs vs Layout Builder — Tableau Décisionnel

| Critère | Paragraphs | Layout Builder |
|---------|-----------|----------------|
| Qui place les composants ? | L'éditeur dans le formulaire de nœud | L'éditeur via l'interface visuelle |
| Structure définie par qui ? | Le développeur (types de paragraphes) | Le développeur (layouts + blocs) ET l'éditeur |
| Mise en page identique sur tous les nœuds du type | ✅ Recommandé | ❌ Complexe à enforcer |
| Mise en page unique par nœud | ⚠️ Possible mais rigide | ✅ Natif |
| Composants réutilisables entre types | ✅ Même paragraph type | ⚠️ Blocs configurables réutilisables |
| Performance | ⚠️ N+1 queries à surveiller | ✅ Meilleure (blocs en cache) |
| Migration de données existantes | ✅ Mature (`entity_reference_revisions`) | ⚠️ Moins d'outils |
| Headless / JSON:API | ✅ Normalisé nativement | ⚠️ Complexe à normaliser |
| Nested content (sections dans sections) | ✅ Possible (risqué) | ✅ Layout sections natif |

**Règle de décision :** Si la mise en page est définie par le dev et que l'éditeur remplit des composants → **Paragraphs**. Si l'éditeur doit personnaliser la mise en page visuellement → **Layout Builder**.

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| Mettre toute la logique dans des nodes | Entités custom pour données non-contenu | Surcharge révisions/path/menu sur des données métier |
| Paragraphs imbriqués > 3 niveaux | Aplatir la structure, reconsidérer Layout Builder | N+1 exponentiel, formulaires inutilisables |
| Taxonomy comme système de navigation | Menu links pour la navigation | API taxonomy pas conçue pour ça |
| Champ Body (text_with_summary) pour texte sans HTML | String ou plain text | Surcharge format texte inutile |
| Entity Reference sans index en base | `indexes:` dans `hook_schema` | Views avec JOIN ultra-lente |
| Paragraphs chargés un par un en preprocess | `loadMultiple()` ou Views avec JOIN | N+1 queries × nombre de paragraphes |
| Layout Builder sur un type de contenu haute fréquence | Paragraphs ou template figé | Layout Builder stocke la config par nœud |
| Vocabulary taxonomy pour des données métier | Custom entity | Taxonomy n'a pas de champs custom natifs |
| Image field sans `alt` obligatoire | Configurer `alt` = required dans le champ | Accessibilité WCAG 2.1 AA |
| Media field pointant vers File (pas Media entity) | Entity Reference vers bundle Media | Pas de Media Library, pas de réutilisation |

## Évolution par Version Majeure

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| Paragraphs (contrib) | ✅ | ✅ | ✅ | ✅ |
| Layout Builder (core) | ✅ expérimental | ✅ stable | ✅ | ✅ |
| Media Library (core) | ❌ | ✅ | ✅ | ✅ |
| Single Directory Components | ❌ | ❌ | ✅ expérimental | ✅ stable |
| Computed fields | ✅ | ✅ | ✅ | ✅ |
| ContentEntityBase | ✅ | ✅ | ✅ | ✅ |
| JSON:API pour Paragraphs | contrib | contrib | contrib | contrib |
| Focal Point | contrib | contrib | contrib | contrib |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Mauvaises décisions architecturales et leurs conséquences.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions.

## See Also

- `drupal-core` — ContentEntityBase, Plugin system, Field API
- `drupal-theming` — Templates Twig pour Paragraphs, Layout Builder, Nodes
- `drupal-views` — Lister et filtrer des entités custom, Paragraphs dans Views
- `drupal-performance` — N+1 queries avec Paragraphs, cache Layout Builder
- `drupal-migration` — Migrer des données vers Paragraphs, depuis Paragraphs
- `drupal-config` — Exporter/importer la config des Content Types et Fields
