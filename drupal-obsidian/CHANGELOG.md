# Changelog — drupal-obsidian

---

## v1.1 — 2026-05-14

**Bugs corrigés :**
- `extraction.md` : `_build_fields_table` lisait `field_type` sur les instances field.field.* qui ne l'ont pas → croise maintenant avec les storages via `all_storages` dict
- `extraction.md` : Labels avec guillemets/apostrophes cassaient le YAML frontmatter → `safe_yaml_str()` ajouté
- `extraction.md` : `sys` importé inutilement et `yaml_link()` morte retirés
- `extraction.md` : Display IDs Views underscore→hyphen non convertis → `drupal_id_to_twig_filename()` appliqué
- `extraction.md` : Chemin personnel hardcodé dans le script Drush → `getenv('OBSIDIAN_VAULT')` avec fallback
- `dataview-queries.md` : `contains(file.outlinks, [[field_image]])` invalide → `file.outlinks.file.path`
- `graphify-integration.md` : `grep -oP` incompatible macOS → version portable POSIX

**Incohérence majeure résolue :**
- Convention wikilinks Twig unifiée partout : `[[tpl-node--article]]` (préfixe `tpl-`, sans `.html.twig`)
  - Fichiers concernés : extraction.md, note-templates.md, wikilinks-mapping.md, vault-structure.md
- Fonctions utilitaires ajoutées : `safe_yaml_str()`, `drupal_id_to_twig_filename()`, `tpl_link()`

**Extraction ajoutée :**
- `extract_view_displays()` — extrait `core.entity_view_display.*` en notes (déclaré "Important" mais manquant)
- Notes générées dans `Displays/` avec composants, formatters, lien vers template Twig

**Nouvelles leçons :**
- Convention templates Twig — décision `tpl-` documentée définitivement
- `_build_fields_table` avec `all_storages` — cause et fix du bug `?`
- `grep -oP` macOS — alternative POSIX portative

---

## v1.0 — 2026-05-14

**Création initiale**

### Couverture

**`SKILL.md`**
- Chaîne complète : `config/sync/*.yml` → Obsidian vault → graphify
- Quick Decision Table (15 entrées)
- Typage des notes (frontmatter standard)
- Anti-patterns critiques (5 entrées)

**`extraction.md`**
- Table de sélection des fichiers YAML à cibler (priorité rouge/orange/jaune)
- Script Python complet (~200 lignes) avec fonctions pour :
  - Content Types (`node.type.*`)
  - Paragraphs (`paragraphs.paragraphs_type.*`)
  - Field Storages (`field.storage.*`)
  - Field Instances (`field.field.*`) — chargement pour les tables
  - Taxonomies (`taxonomy.vocabulary.*`)
  - Views (`views.view.*`)
- Helpers : `_build_fields_table()`, `_list_modules()`, wikilinks automatiques
- Script Drush alternatif (PHP inline)
- Commandes d'utilisation

**`vault-structure.md`**
- Arborescence complète du vault (9 dossiers)
- Paramètres Obsidian recommandés (Files & Links, Core Plugins, Community Plugins)
- Config `.obsidian/graph.json` pour colorisation par tag
- Tags standards pour le filtrage Graph View
- Note d'index `Dashboard/Architecture Overview.md` avec requêtes Dataview intégrées
- Convention de nommage des notes par type

**`note-templates.md`**
- 6 templates complets avec frontmatter YAML :
  - Content Type : table des champs, templates Twig, modes d'affichage, hooks, services
  - Paragraph : champs, templates, entités parentes
  - Champ (Field Storage) : usage dans les bundles, settings, formatters, risque architectural
  - Template Twig : variables disponibles, mapping config, preprocess
  - Hook PHP : bundles ciblés, variables injectées, templates impactés
  - View : displays, entités exposées, filtres

**`wikilinks-mapping.md`**
- Explication du mécanisme backlinks Obsidian
- 7 stratégies de liaison : Entités→Champs, Entités→Templates, Templates→Config, Templates→Preprocess, Logic→Entités, Views→Entités, Entity References
- Comment lire le Graph View pour identifier les nœuds critiques
- Filtrage du graphe par dossier + colorisation par tag
- Patterns avancés : hooks conditionnels, modes d'affichage, workflows
- Note spéciale "Entity Reference Map" pour cartographier toutes les relations ER

**`dataview-queries.md`**
- 20 requêtes Dataview organisées par catégorie :
  - Vue d'ensemble (2 requêtes de comptage)
  - Audit des manques (6 requêtes : paragraphs sans Twig, CT sans Twig, champs orphelins, hooks sans cible)
  - Cartographie des relations (5 requêtes : ER→Taxonomies, ER→Paragraphs, champs critiques, entities utilisant un champ, mapping Template→Config)
  - Analyse des risques (3 requêtes : entités complexes, hooks multi-bundles, views sans template)
  - Suivi de la documentation (3 requêtes : notes périmées, nouvelles notes, couverture)
  - Avancé (2 requêtes complexes + DataviewJS complet)

**`graphify-integration.md`**
- Pourquoi graphify sur un vault Drupal (vs Graph View natif)
- Stratégies de filtrage pré-graphify (sous-ensembles ≤ 50 notes)
- Ce que graphify fait : nœuds = notes, arêtes = wikilinks, communautés, centralité
- Interprétation des communautés attendues (Cluster Article, Cluster Paragraphs, etc.)
- Format du rapport JSON graphify annoté
- Template Canvas Obsidian complet (JSON) pour flux Config → Twig
- Workflow complet : drush cex → extraction → Obsidian → graphify → insights
- Sous-ensembles thématiques : "impact d'une migration de champ", "périmètre de refactoring"

**`lessons.md`**
- 7 leçons pré-remplies :
  - Frontmatter YAML incompatible avec Dataview (guillemets et caractères spéciaux)
  - Wikilinks vers notes inexistantes (faux orphelins dans Graph View)
  - Graphify sur tout le vault (graphe illisible)
  - Encodage YAML avec allow_unicode
  - `file.outlinks` vs `file.inlinks` confondus dans Dataview
  - Vault non synchronisé avec config Drupal
  - Nommage notes de templates (double extension)

---

## Compatibilité

| Skill version | Drupal | Obsidian | Dataview plugin | Python |
|--------------|--------|----------|----------------|--------|
| v1.0 | D9, D10, D11 | 1.x | 0.5+ | 3.8+ |
