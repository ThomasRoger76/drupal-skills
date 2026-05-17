# Structure du Vault Obsidian

## Architecture des Dossiers

```
📁 Drupal Architecture/          ← Racine du vault (ou sous-dossier dans un vault existant)
│
├── 📁 Entities/                 ← Toutes les entités Drupal
│   ├── 📁 ContentTypes/         ← node.type.*
│   │   ├── article.md
│   │   ├── page.md
│   │   └── projet.md
│   ├── 📁 Paragraphs/           ← paragraphs.paragraphs_type.*
│   │   ├── hero.md
│   │   ├── card.md
│   │   └── slider.md
│   ├── 📁 Taxonomies/           ← taxonomy.vocabulary.*
│   │   ├── tags.md
│   │   └── categories.md
│   └── 📁 Media/                ← media.type.*
│       ├── image.md
│       └── document.md
│
├── 📁 Fields/                   ← field.storage.* (champs réutilisables)
│   ├── field_image.md
│   ├── field_tags.md
│   ├── field_body.md
│   └── field_couleur.md
│
├── 📁 Theme/                    ← Thème et templates
│   ├── 📁 Templates/            ← Templates Twig (créés manuellement)
│   │   ├── node--article.html.twig.md
│   │   ├── paragraph--hero.html.twig.md
│   │   └── page.html.twig.md
│   ├── 📁 Libraries/            ← .libraries.yml
│   │   └── global-styling.md
│   └── 📁 Preprocess/          ← Fonctions preprocess du .theme
│       └── preprocess_node.md
│
├── 📁 Logic/                    ← Code PHP custom
│   ├── 📁 Hooks/                ← Hooks dans .module
│   │   ├── hook_preprocess_node.md
│   │   ├── hook_entity_access.md
│   │   └── hook_form_alter.md
│   ├── 📁 Services/             ← Services Drupal custom
│   │   └── article_service.md
│   └── 📁 EventSubscribers/     ← Event subscribers
│       └── mon_subscriber.md
│
├── 📁 Views/                    ← views.view.*
│   ├── mes_articles.md
│   └── page_accueil.md
│
├── 📁 Workflows/                ← workflows.workflow.*
│   └── editorial.md
│
└── 📁 Dashboard/               ← Notes d'index et de synthèse
    ├── Architecture Overview.md  ← Vue d'ensemble (Canvas)
    ├── Champs critiques.md       ← Dataview : champs les plus utilisés
    ├── Templates manquants.md    ← Dataview : paragraphs sans Twig
    └── Entity Reference Map.md   ← Dataview : toutes les relations ER
```

---

## Configuration Obsidian Recommandée

### Paramètres essentiels

```
Settings → Files & Links:
  ✅ Use [[Wikilinks]]
  ✅ Default location for new notes → Dans le dossier courant
  ✅ New link format → Shortest path

Settings → Core Plugins:
  ✅ Backlinks → Activé + affichage dans document
  ✅ Graph view → Activé
  ✅ Tags → Activé

Settings → Community Plugins (installer) :
  - Dataview (obligatoire pour les requêtes)
  - Graph Analysis (pour les métriques de centralité)
  - Kanban (optionnel — pour le suivi des tâches d'archi)
```

### `.obsidian/graph.json` — Paramètres Graph View

```json
{
  "colorGroups": [
    {"query": "tag:#entity",       "color": {"a": 1, "rgb": 16711680}},
    {"query": "tag:#paragraph",    "color": {"a": 1, "rgb": 16744192}},
    {"query": "tag:#field",        "color": {"a": 1, "rgb": 65280}},
    {"query": "tag:#twig-template","color": {"a": 1, "rgb": 255}},
    {"query": "tag:#hook",         "color": {"a": 1, "rgb": 9109504}},
    {"query": "tag:#view",         "color": {"a": 1, "rgb": 8388736}}
  ],
  "showTags": false,
  "showAttachments": false,
  "hideUnresolved": false,
  "nodeSize": 5,
  "lineSizeMultiplier": 1,
  "centerStrength": 0.518713248970312,
  "repelStrength": 10,
  "linkStrength": 1
}
```

---

## Tags Standards — Filtrage dans Graph View

Chaque note doit avoir ces tags pour que le Graph soit coloré correctement :

```markdown
---
tags: [entity, node, content-type]          ← Content Types
tags: [entity, paragraph]                   ← Paragraphs
tags: [entity, taxonomy]                    ← Vocabulaires
tags: [field, image]                        ← Champs (+ type du champ)
tags: [twig-template, node]                 ← Templates Twig
tags: [hook, preprocess]                    ← Hooks PHP
tags: [service]                             ← Services
tags: [view]                                ← Views
tags: [workflow]                            ← Workflows
---
```

---

## Note d'Index — `Dashboard/Architecture Overview.md`

```markdown
---
type: dashboard
tags: [dashboard, index]
---

# Architecture — [Nom du Projet]

## Vue d'ensemble

> Ce vault documente l'architecture Drupal du projet [Nom].
> Généré le : [DATE] depuis `config/sync/`

## Statistiques

```dataview
TABLE length(rows.file.name) as "Nombre"
FROM ""
GROUP BY type
SORT type ASC
```

## Content Types

```dataview
LIST
FROM "Entities/ContentTypes"
SORT file.name ASC
```

## Paragraphs

```dataview
LIST
FROM "Entities/Paragraphs"
SORT file.name ASC
```

## Champs les Plus Utilisés (Top 10)

Voir [[Champs critiques]]

## Templates Manquants

Voir [[Templates manquants]]

## Liens rapides
- [[Entities/ContentTypes/article]] — Content Type principal
- [[Logic/Hooks/hook_preprocess_node]] — Preprocess principal
- [[Views/mes_articles]] — View principale
```

---

## Naming Convention des Notes

| Type | Convention | Exemple |
|------|-----------|---------|
| Content Type | `{bundle}.md` | `article.md` |
| Paragraph | `{bundle}.md` | `hero.md` |
| Champ storage | `{field_name}.md` | `field_image.md` |
| Template Twig | `tpl-{template-name}.md` (préfixe `tpl-`, SANS `.html.twig`) | `tpl-node--article.md` |
| Hook | `hook_{nom}.md` | `hook_preprocess_node.md` |
| View | `{view_id}.md` | `mes_articles.md` |
| Mode d'affichage | `display.{entity}.{bundle}.{mode}.md` | `display.node.article.full.md` |
| Service | `{service_id}.md` | `article_service.md` |

**Règle :** pas d'espaces dans les noms de fichiers — utiliser `_` ou `-` selon la convention Drupal du type.

> **Convention templates Twig :** préfixe `tpl-` + tirets (pas d'underscores ni de `.html.twig` dans le nom).  
> `node--article.html.twig` → note `tpl-node--article.md` → wikilink `[[tpl-node--article]]`  
> Raison : les doubles extensions (`.html.twig.md`) causent des problèmes dans certains OS et éditeurs.

---

## Workflow Git pour le Vault en Équipe

### `.gitignore` pour un vault Obsidian versionné

```gitignore
# .gitignore du vault Obsidian
# Ignorer l'état de l'UI (onglets ouverts, position scroll)
.obsidian/workspace.json
# Ignorer la position des nœuds dans le graph (change à chaque session)
.obsidian/graph.json
# Ignorer les plugins installés localement (chaque dev les installe lui-même)
.obsidian/plugins/

# GARDER dans git :
# .obsidian/app.json              → préférences éditeur partagées
# .obsidian/community-plugins.json → liste des plugins requis (pas leurs fichiers)
# .obsidian/core-plugins.json     → plugins core activés
# .obsidian/hotkeys.json          → raccourcis clavier d'équipe (optionnel)
```

### Convention de commit pour les mises à jour de vault

```bash
# Format recommandé pour les commits du vault
git add docs/architecture/
git commit -m "docs: update vault — add Content Type X, update Field Y"

# Ignorer les conflits sur workspace.json en favorisant toujours la version locale
git config merge.ours.driver true
echo '.obsidian/workspace.json merge=ours' >> .gitattributes
```

### Mise à jour automatique du vault via CI/CD

```yaml
# .gitlab-ci.yml — job de mise à jour du vault après deploy
update-vault:
  stage: post-deploy
  image: python:3.11-slim
  script:
    - pip install pyyaml
    - python3 scripts/drupal_to_obsidian.py
        --config config/sync
        --vault docs/architecture
        --types content_type,paragraph,view,migration,workflow,sdc_component
    - git config user.email "ci@monprojet.com"
    - git config user.name "CI Bot"
    - git add docs/architecture/
    - git diff --cached --quiet || git commit -m "docs: update architecture vault [skip ci]"
    - git push
  rules:
    # Déclencher uniquement quand des fichiers config YAML changent
    - changes:
        - "config/**/*.yml"
  only:
    - main
```

> **Astuce :** le `[skip ci]` dans le message de commit empêche une boucle CI infinie quand le bot pousse le vault mis à jour.

### Résoudre les conflits de merge dans le vault

```bash
# Conflits fréquents sur les notes auto-générées : accepter toujours la version CI
git checkout --theirs docs/architecture/Entities/
git add docs/architecture/Entities/

# Pour les notes manuelles (templates, dashboard) : résoudre manuellement
git checkout --ours docs/architecture/Dashboard/
git add docs/architecture/Dashboard/
```
