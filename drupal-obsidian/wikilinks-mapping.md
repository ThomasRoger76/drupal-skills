# Mapping Relationnel — Wikilinks & Backlinks

## Le Principe des Backlinks dans Obsidian

Quand la note `field_image.md` est liée depuis `article.md` via `[[field_image]]`, Obsidian crée automatiquement un **backlink** dans `field_image.md` qui liste tous les bundles l'utilisant.

```
article.md → [[field_image]]
projet.md  → [[field_image]]
hero.md    → [[field_image]]

↓ Dans field_image.md, le panneau Backlinks affiche :
← article.md (1 mention)
← projet.md  (1 mention)
← hero.md    (1 mention)

↓ Dans Graph View, field_image devient un nœud avec 3 connexions
  → Nœud critique visible instantanément
```

---

## Stratégie de Liaison — Carte des Relations

### 1. Entités → Champs

Dans chaque note d'entité (ContentType, Paragraph), lier vers les notes de champs :

```markdown
# article.md
## Champs
| Champ | Type | Référence |
|-------|------|-----------|
| `field_image` | image | [[field_image]] |    ← Crée backlink dans field_image.md
| `field_tags` | entity_ref | [[field_tags]] |    ← Crée backlink dans field_tags.md
| `body` | text | [[field_body]] |
```

### 2. Entités → Templates Twig

```markdown
# article.md
## Templates Twig
- [[tpl-node--article]]        ← Backlink dans le template (convention : préfixe tpl-)
- [[tpl-node--article--teaser]]
```

### 3. Templates → Config d'affichage + Champs

```markdown
# tpl-node--article.md   ← Nom de fichier avec préfixe tpl- (pas de .html.twig)
## Mapping avec la config
- **Config :** [[display.node.article.default]]   ← Lie le template à sa config
- [[field_image]] utilisé via `{{ content.field_image }}`
- [[field_tags]] utilisé via `{{ content.field_tags }}`
```

### 4. Templates → Preprocess (Logic → Theme)

```markdown
# node--article.html.twig.md
## Variables injectées depuis le preprocess
- `date_formatted` — voir [[hook_preprocess_node]]
- `is_author` — voir [[hook_preprocess_node]]
```

### 5. Logic → Entités (Hooks → Content Types)

```markdown
# hook_preprocess_node.md
## Bundles ciblés
- [[article]] — Ajoute `date_formatted`, `is_author`
- [[projet]] — Ajoute `categorie_label`
```

### 6. Entités → Entités (Entity References)

```markdown
# article.md
## Paragraphs imbriqués
- `field_contenu` → [[hero]], [[card]], [[slider]]

## Références taxonomies
- `field_tags` → [[tags]] (vocabulaire)
- `field_categorie` → [[categories]] (vocabulaire)
```

### 7. Views → Entités exposées

```markdown
# mes_articles.md
## Entités exposées
- [[article]] — Bundle source de la View
```

---

## Le Graph View — Lire le Graphe Drupal

Après avoir lié toutes les notes, le Graph View révèle la topologie de l'architecture :

### Identifier les nœuds critiques

**Un nœud avec de nombreuses connexions = un point de rupture.**

```
field_image ──── article
    │          ├── projet
    │          ├── hero (paragraph)
    │          └── portfolio

→ field_image est connecté à 4 entités
→ Toute modification de ce champ impacte 4 bundles
→ = Nœud critique à documenter en priorité
```

### Filtrer le graphe par dossier

Dans Graph View → Filtres :
- Afficher uniquement `Entities/` → voir la topologie des entités
- Afficher uniquement `Fields/` → voir les champs orphelins
- Afficher `Entities/ + Theme/` → voir le mapping Config → Twig
- Afficher `Theme/ + Logic/` → voir le mapping Twig ↔ Hooks

### Colorisation par tag

Configurer dans Graph View → Groupes de couleurs :
- `tag:#entity` → Rouge (Content Types, Paragraphs)
- `tag:#field` → Vert (Champs)
- `tag:#twig-template` → Bleu (Templates)
- `tag:#hook` → Violet (Logic)
- `tag:#view` → Orange (Views)

---

## Patterns de Liaison Avancés

### Liaison Conditionnelle — Hooks qui ne s'appliquent qu'à certains bundles

```markdown
# hook_form_node_article_form_alter.md
## Portée
- Appliqué UNIQUEMENT au formulaire de : [[article]]
- Impact : Ajoute le champ `field_date` comme requis

## Formulaires impactés
- `node_article_form` (création)
- `node_article_edit_form` (édition)
```

### Liaison Mode d'Affichage → Template

```markdown
# display.node.article.teaser.md
---
type: view_display
entity_type: node
bundle: article
view_mode: teaser
---

# Mode d'affichage : Article — Teaser

## Template actif
- [[tpl-node--article--teaser]]

## Champs configurés
| Champ | Formatter | Settings |
|-------|-----------|---------|
| `title` | Lien | — |
| `field_image` | Responsive Image | Style: [[article-teaser]] |
| `field_tags` | Entity Reference Label | — |
| `body` | Résumé | 150 caractères |
```

### Liaison Paragraph → ContentType (usage réel)

Documenter MANUELLEMENT qui utilise quel Paragraph :

```markdown
# hero.md (Paragraph)
## Utilisé dans ces bundles
- [[article]] → champ `field_hero_section` (mode : défaut)
- [[page]] → champ `field_sections` (mode : défaut)
```

### Liaison Workflow → Entité

```markdown
# editorial.md (Workflow)
## Bundles soumis à ce workflow
- [[article]]
- [[projet]]

## États
- `draft` → `review` → `published` → `archived`

## Transitions
| De | Vers | Permission |
|----|------|-----------|
| draft | review | `use editorial transition review` |
| review | published | `use editorial transition publish` |
```

---

## Note Spéciale — `Entity Reference Map`

Créer une note dédiée pour mapper toutes les relations Entity Reference :

```markdown
# Entity Reference Map
---
type: dashboard
---

# Carte des Entity References

## Node → Paragraph
| Bundle | Champ | Paragraphs autorisés |
|--------|-------|---------------------|
| [[article]] | `field_contenu` | [[hero]], [[card]], [[slider]] |
| [[page]] | `field_sections` | [[hero]], [[card]], [[text_block]] |
| [[projet]] | `field_blocs` | [[card]], [[gallery]] |

## Node → Taxonomy
| Bundle | Champ | Vocabulaire |
|--------|-------|------------|
| [[article]] | `field_tags` | [[tags]] |
| [[article]] | `field_categorie` | [[categories]] |
| [[projet]] | `field_type` | [[types_projet]] |

## Node → Media
| Bundle | Champ | Types Media |
|--------|-------|------------|
| [[article]] | `field_image` | image |
| [[projet]] | `field_galerie` | image (multiple) |

## Paragraph → Taxonomy
| Paragraph | Champ | Vocabulaire |
|-----------|-------|------------|
| [[card]] | `field_categorie` | [[categories]] |
```
