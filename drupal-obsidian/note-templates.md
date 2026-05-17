# Templates de Notes — Un par Type Drupal

## Template : Content Type

```markdown
---
type: content_type
bundle: article
drupal_type: node
label: "Article"
machine_name: article
revision: true
display_submitted: true
tags: [entity, node, content-type]
created: 2026-05-14
updated: 2026-05-14
---

# Article

> [!info] Identifiants
> **Machine name :** `article` | **Drupal type :** `node`

## Champs
| Champ | Type | Requis | Traduit | Référence |
|-------|------|--------|---------|-----------|
| `title` | `string` | ✅ | ✅ | (core) |
| `body` | `text_with_summary` | ❌ | ✅ | [[field_body]] |
| `field_image` | `image` | ❌ | ❌ | [[field_image]] |
| `field_tags` | `entity_reference` → [[tags]] | ❌ | ❌ | [[field_tags]] |

## Templates Twig
- [[tpl-node--article]]
- [[tpl-node--article--full]]
- [[tpl-node--article--teaser]]

## Modes d'affichage
- [[display.node.article.default]]
- [[display.node.article.teaser]]
- [[display.node.article.search_index]]

## Logic
### Hooks custom
- [[hook_preprocess_node]] (si `$node->bundle() === 'article'`)
- [[hook_entity_access]] (règles d'accès custom)

### Services utilisant ce bundle
- [[article_service]]

## Paragraphs imbriqués
> Champs de type entity_reference vers Paragraphs :
- `field_contenu` → [[hero]], [[card]], [[slider]]

## Vues qui exposent ce bundle
- [[mes_articles]]
- [[frontpage]]

## Notes
> Zone libre pour les remarques d'architecture, décisions, dette technique...
```

---

## Template : Paragraph

```markdown
---
type: paragraph
bundle: hero
drupal_type: paragraph
label: "Hero Banner"
machine_name: hero
tags: [entity, paragraph]
created: 2026-05-14
---

# Paragraph : Hero Banner

> [!info] Identifiants
> **Machine name :** `hero` | **Plugin :** `paragraphs`

## Champs
| Champ | Type | Requis | Référence |
|-------|------|--------|-----------|
| `field_titre` | `string` | ✅ | [[field_titre]] |
| `field_image` | `image` | ✅ | [[field_image]] |
| `field_cta_lien` | `link` | ❌ | [[field_cta_lien]] |
| `field_sous_titre` | `string_long` | ❌ | [[field_sous_titre]] |

## Templates Twig
- [[tpl-paragraph--hero]]
- [[tpl-paragraph--hero--default]]

## Mode d'affichage
- [[display.paragraph.hero.default]]

## Utilisé dans ces bundles
> Entités qui ont un champ entity_reference vers ce type :
- [[article]] → `field_contenu`
- [[page]] → `field_sections`

## Preprocess
- [[hook_preprocess_paragraph]] (si `$paragraph->bundle() === 'hero'`)
```

---

## Template : Champ (Field Storage)

```markdown
---
type: field_storage
field_name: field_image
entity_type: node
field_type: image
cardinality: 1
tags: [field, image]
created: 2026-05-14
---

# Champ : `field_image`

> [!info] Type
> `image` | Cardinalité : 1 | Entity type : `node`

## Utilisé dans ces bundles
> Voir les **Backlinks** d'Obsidian pour la liste complète automatique.
> Bundles connus :
- [[article]] — Image principale
- [[projet]] — Image de couverture
- [[hero]] — Image de fond (paragraph)

## Settings
- Extensions autorisées : `png gif jpg jpeg webp`
- Taille max : `2 MB`
- Résolution max : `2000×2000`
- Alt text requis : ✅

## Formatter utilisé par défaut
- **Full :** `image` (Responsive Image Style: [[hero-image]])
- **Teaser :** `image` (Responsive Image Style: [[article-teaser]])

## Risque architectural
> [!warning] Champ critique
> Ce champ est utilisé dans X bundles — tout changement de type impacte l'ensemble.

## Templates Twig associés
- [[tpl-field--field-image]]
- [[tpl-field--node--field-image]]
```

---

## Template : Template Twig

```markdown
---
type: twig_template
template_name: node--article
entity_type: node
bundle: article
view_mode: all
tags: [twig-template, node]
created: 2026-05-14
---

# Template : `node--article.html.twig`

> [!info] Suggestion
> Appliqué à tous les modes d'affichage du bundle `article`

## Variables disponibles (depuis preprocess)
| Variable | Type | Source |
|----------|------|--------|
| `{{ node }}` | NodeInterface | Core |
| `{{ content }}` | render array | Core |
| `{{ label }}` | string | Core |
| `{{ url }}` | string | Core |
| `{{ date_formatted }}` | string | [[hook_preprocess_node]] |
| `{{ is_author }}` | bool | [[hook_preprocess_node]] |

## Champs affichés
- `{{ content.field_image }}` — [[field_image]]
- `{{ content.field_tags }}` — [[field_tags]]
- `{{ content\|without('field_image', 'field_tags') }}` — Reste du contenu

## Mapping avec la config
- **Config d'affichage :** [[display.node.article.default]]
- **Libraries CSS/JS :** [[mon_theme/article-styles]]

## Entité parente
- [[article]]

## Preprocess associé
- [[hook_preprocess_node]] (section article)

## Suggestions plus spécifiques
- [[tpl-node--article--full]] (surcharge du mode full)
- [[tpl-node--article--teaser]] (surcharge du mode teaser)

## Notes
> Dernière modification : [DATE]
> Raison : [Description du changement]
```

---

## Template : Hook PHP

```markdown
---
type: hook
hook_name: hook_preprocess_node
module: mon_module
file: mon_module.module
tags: [hook, preprocess]
created: 2026-05-14
---

# Hook : `hook_preprocess_node`

> **Fichier :** `web/modules/custom/mon_module/mon_module.module`

## Ce que ce hook fait
Enrichit les variables Twig pour les nœuds avant rendu.

## Bundles ciblés
- [[article]] — Ajoute `date_formatted`, `is_author`
- [[projet]] — Ajoute `categorie_label`

## Variables injectées
| Variable | Type | Bundle | Usage |
|----------|------|--------|-------|
| `date_formatted` | string | article | Affichage formaté `{{date_formatted}}` |
| `is_author` | bool | article | Classe CSS `node--own` |
| `categorie_label` | string | projet | Label de la catégorie |

## Templates impactés
- [[tpl-node--article]]
- [[tpl-node--projet]]

## Extrait de code
```php
function mon_module_preprocess_node(array &$variables): void {
  $node = $variables['node'];
  if ($node->bundle() === 'article') {
    $variables['date_formatted'] = \Drupal::service('date.formatter')
      ->format($node->getCreatedTime(), 'custom', 'd/m/Y');
  }
}
```

---

## Template : Migration

```markdown
---
type: migration
migration_id: import_articles
source_plugin: csv
destination_plugin: entity:node
bundle: article
langcode: fr
status: complete
last_run: 2026-05-15
items_imported: 1250
items_failed: 3
tags: [migration]
created: 2026-05-15
---

# Migration : Import Articles

> [!info] Identifiants
> **ID :** `import_articles` | **Status :** `complete`

## Source
- Plugin : `csv`
- Fichier : `data/articles.csv`
- Champs source : `id`, `titre`, `corps`, `created`

## Processus
| Source | Destination | Plugin process |
|--------|-------------|----------------|
| `titre` | `title` | direct |
| `corps` | `body/value` | direct |
| `categorie_id` | `field_categorie/target_id` | `migration_lookup` → [[Migration Import Categories]] |
| `image_id` | `field_image/target_id` | `migration_lookup` → [[Migration Import Images]] |

## Destination
- Entité : [[Content Type Article]]
- Bundle : `article`
- Langcode : `fr`

## Dépendances
> Ces migrations doivent tourner **avant** celle-ci :
- [[Migration Import Categories]] (`required`)
- [[Migration Import Images]] (`required`)

## Commandes
```bash
docker compose exec php drush migrate:import import_articles
docker compose exec php drush migrate:status import_articles
docker compose exec php drush migrate:rollback import_articles
```

## Notes
> Zone libre : problèmes connus, mapping spécial, performances...
```

---

## Template : Workflow (Content Moderation)

```markdown
---
type: workflow
workflow_id: editorial
entity_types: [node]
bundles: [article, page]
states: [draft, needs_review, published, archived]
tags: [workflow]
created: 2026-05-15
---

# Workflow : Editorial

> [!info] Identifiants
> **Machine name :** `editorial` | **Module :** `content_moderation`

## États
| État | Label | Par défaut | Publié |
|------|-------|-----------|--------|
| `draft` | Brouillon | ✅ | ❌ |
| `needs_review` | En révision | ❌ | ❌ |
| `published` | Publié | ❌ | ✅ |
| `archived` | Archivé | ❌ | ❌ |

## Transitions
| De | Vers | Label | Permission |
|----|------|-------|-----------|
| `draft` | `needs_review` | Soumettre | `use editorial transition submit_for_review` |
| `needs_review` | `published` | Publier | `use editorial transition publish` |
| `needs_review` | `draft` | Renvoyer en brouillon | `use editorial transition back_to_draft` |
| `published` | `archived` | Archiver | `use editorial transition archive` |

## Content Types concernés
> Entités soumises à ce workflow :
- [[Content Type Article]]
- [[Content Type Page]]

## Rôles et permissions
| Rôle | Transitions autorisées |
|------|----------------------|
| `editor` | submit_for_review |
| `reviewer` | publish, back_to_draft |
| `administrator` | toutes |

## Hooks liés
- [[hook_content_moderation_state_transition]] (si logique custom sur transition)
```

---

## Template : SDC Component

```markdown
---
type: sdc_component
component_id: mon_theme:card
theme: mon_theme
directory: components/card/
props: [title, body, image_url, variant]
slots: [footer]
tags: [sdc, component]
created: 2026-05-15
---

# SDC : Card

> [!info] Identifiants
> **ID :** `mon_theme:card` | **Répertoire :** `components/card/`

## Props
| Prop | Type | Requis | Description |
|------|------|--------|-------------|
| `title` | `string` | ✅ | Titre de la carte |
| `body` | `string` | ❌ | Description ou résumé |
| `image_url` | `string` | ❌ | URL de l'image |
| `variant` | `string` (enum) | ❌ | `default` / `featured` / `compact` |

## Slots
| Slot | Description |
|------|-------------|
| `footer` | Contenu optionnel du pied de carte (liens, boutons) |

## Usage Twig
```twig
{% raw %}
{% include 'mon_theme:card' with {
  title: node.title.value,
  body: content.body,
  image_url: file_url(node.field_image.entity.fileUri),
  variant: 'featured',
} %}
{% endraw %}
```

## Utilisé par
> Content Types et templates qui embedent ce composant :
- [[Content Type Article]] (mode teaser)
- [[Content Type Page]] (section cards)

## Template associé
- [[tpl-node--article--teaser]]

## Notes
> Accessibilité : le slot `footer` doit contenir un lien avec aria-label si l'image est décorative.
```

---

## Template : View

```markdown
---
type: view
view_id: mes_articles
label: "Mes Articles"
base_table: node_field_data
entity_type: node
bundle: article
tags: [view]
created: 2026-05-14
---

# View : Mes Articles

## Displays
| Display | Plugin | Path/Block | Template |
|---------|--------|-----------|---------|
| `page_1` | Page | `/mes-articles` | [[tpl-views-view--mes-articles--page-1]] |
| `block_1` | Block | (région sidebar) | [[tpl-views-view--mes-articles--block-1]] |

## Entités exposées
- [[article]] — Bundle source

## Filtres
| Filtre | Type | Exposé | Lien |
|--------|------|--------|------|
| `status` | boolean | ❌ | — |
| `type` | string | ❌ | — |
| `field_tags` | entity_reference | ✅ | [[field_tags]] |

## Champs affichés
- `title` → lien canonique
- `field_image` → [[field_image]]
- `field_tags` → [[field_tags]]

## Hooks liés
- [[hook_views_data_alter]] (si champs custom exposés)
```
