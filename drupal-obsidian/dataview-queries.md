# Dataview — Requêtes d'Audit Architectural

## Prérequis

Le plugin **Dataview** doit être installé dans Obsidian (Community Plugins). Les requêtes fonctionnent grâce aux frontmatter YAML de chaque note générée par le script d'extraction.

---

## Requêtes Essentielles — Dashboard Architecture

### 1. Vue d'ensemble — Comptage par type

```dataview
TABLE length(rows.file.name) as "Nombre de notes"
FROM ""
WHERE type != null
GROUP BY type
SORT length(rows.file.name) DESC
```

### 2. Tous les Content Types avec leur label

```dataview
TABLE label as "Label", bundle as "Machine name", tags as "Tags"
FROM "Entities/ContentTypes"
WHERE type = "content_type"
SORT label ASC
```

### 3. Tous les Paragraphs avec leur label

```dataview
TABLE label as "Label", bundle as "Machine name"
FROM "Entities/Paragraphs"
WHERE type = "paragraph"
SORT label ASC
```

---

## Requêtes d'Audit — Trouver les Manques

### 4. Paragraphs SANS template Twig associé

Détecte les Paragraphs qui n'ont pas de note `type: twig_template` avec un backlink (lien entrant) vers ce paragraph.

> **Logique correcte :** on cherche dans les notes de type `twig_template` si l'une d'elles référence (via outlink) la note du paragraph — pas si la note du paragraph a un outlink vers un template.

```dataviewjs
// Paragraphs sans template Twig associé
// Un template Twig "couvre" un paragraph si sa note contient un wikilink vers ce paragraph
dv.table(
  ["Paragraph", "Bundle", "Nb champs"],
  dv.pages('#paragraph')
    .where(p =>
      !dv.pages('#twig_template')
        .some(t => t.file.outlinks.some(l => l.path === p.file.path))
    )
    .map(p => [p.file.link, p.bundle, p.fields?.length ?? 0])
)
```

> [!warning] Ces Paragraphs utilisent le template générique `paragraph.html.twig`
> Ils peuvent manquer de personnalisation visuelle.

> **Pourquoi cette approche ?** Les outlinks d'un paragraph pointent vers ses dépendances (champs, entités parentes), pas vers ses templates. La relation de couverture va dans l'autre sens : c'est la note de template qui référence le paragraph qu'elle implémente.

### 5. Content Types SANS template Twig de base

```dataviewjs
// Content Types sans template Twig associé
dv.table(
  ["Content Type", "Bundle", "Nb connexions"],
  dv.pages('#content_type')
    .where(ct =>
      !dv.pages('#twig_template')
        .some(t => t.file.outlinks.some(l => l.path === ct.file.path))
    )
    .map(ct => [ct.file.link, ct.bundle, ct.file.outlinks.length])
)
```

### 6. Champs définis mais non liés à aucune entité (orphelins)

Champs qui n'ont aucun backlink depuis les entités = potentiellement inutilisés.

```dataview
TABLE file.inlinks as "Utilisé par"
FROM "Fields"
WHERE type = "field_storage"
  AND length(file.inlinks) = 0
SORT file.name ASC
```

### 7. Hooks PHP sans entité cible documentée

```dataview
TABLE file.name as "Hook", file.inlinks as "Entités documentées"
FROM "Logic/Hooks"
WHERE type = "hook"
  AND length(file.inlinks) = 0
SORT file.name ASC
```

---

## Requêtes de Cartographie — Comprendre les Relations

### 8. Champs Entity Reference pointant vers des Taxonomies

Requête clé : révèle toutes les relations entité → vocabulaire de taxonomie.

```dataview
TABLE
  file.link as "Note",
  bundle as "Bundle source",
  field_name as "Champ"
FROM ""
WHERE type = "field_storage"
  AND field_type = "entity_reference"
  AND contains(file.content, "taxonomy_term")
SORT bundle ASC
```

### 9. Champs Entity Reference pointant vers des Paragraphs

```dataview
TABLE
  bundle as "Bundle parent",
  field_name as "Champ"
FROM ""
WHERE type = "field_storage"
  AND field_type = "entity_reference"
  AND contains(file.content, "paragraph")
SORT bundle ASC
```

### 10. Champs les Plus Utilisés (Top 10 par nombre de backlinks)

**Les nœuds avec le plus de backlinks = les champs les plus critiques.**

```dataview
TABLE length(file.inlinks) as "Utilisé par N bundles", field_type as "Type"
FROM "Fields"
WHERE type = "field_storage"
SORT length(file.inlinks) DESC
LIMIT 10
```

### 11. Entités utilisant un champ spécifique

Remplacer `field_image` par n'importe quel field_name :

```dataview
TABLE file.link as "Entité", type as "Type"
FROM ""
WHERE contains(file.outlinks.file.path, "Fields/field_image")
SORT type ASC
```

> Note : `file.outlinks` est une liste d'objets — utiliser `.file.path` pour obtenir la chaîne de chemin comparable.

### 12. Mapping complet Template → Config

```dataview
TABLE
  entity_type as "Type d'entité",
  bundle as "Bundle",
  view_mode as "Mode"
FROM "Theme/Templates"
WHERE type = "twig_template"
SORT entity_type ASC, bundle ASC
```

---

## Requêtes de Risque — Identifier les Points de Rupture

### 13. Entités complexes (beaucoup de champs → risque de régression)

```dataview
TABLE
  label as "Label",
  length(file.outlinks) as "Nb connexions (liens sortants)"
FROM "Entities"
WHERE type = "content_type" OR type = "paragraph"
SORT length(file.outlinks) DESC
LIMIT 10
```

### 14. Hooks impactant plusieurs bundles

```dataview
TABLE
  file.name as "Hook",
  length(file.outlinks) as "Bundles ciblés"
FROM "Logic/Hooks"
WHERE type = "hook"
  AND length(file.outlinks) > 1
SORT length(file.outlinks) DESC
```

### 15. Views sans template Twig personnalisé

```dataview
TABLE
  label as "Label",
  view_id as "ID"
FROM "Views"
WHERE type = "view"
SORT label ASC
```

---

## Requêtes de Workflow — Suivi de la Documentation

### 16. Notes sans date de mise à jour (à auditer)

```dataview
TABLE file.name as "Note", created as "Créée le"
FROM ""
WHERE type != null
  AND updated = null
  AND type != "dashboard"
SORT created ASC
```

### 17. Notes créées cette semaine (nouvelles entités)

```dataview
TABLE file.name as "Note", type as "Type", created as "Date"
FROM ""
WHERE type != null
  AND date(created) >= date(today) - dur(7 days)
SORT created DESC
```

### 18. Tableau récapitulatif — Couverture de la documentation

```dataview
TABLE
  length(rows.file.name) as "Notes",
  length(filter(rows.file.name, (x) => contains(x, "html.twig"))) as "Avec template"
FROM "Entities"
GROUP BY type
```

---

## Requêtes Avancées — Analyse en Profondeur

### 19. Chaîne complète : ContentType → Champs → Templates

```dataview
TABLE
  file.link as "ContentType",
  length(file.outlinks) as "Nb champs documentés",
  length(filter(file.outlinks, (x) => contains(x.path, "Theme"))) as "Liens vers templates"
FROM "Entities/ContentTypes"
WHERE type = "content_type"
SORT file.name ASC
```

### 20. Paragraphs utilisés dans plusieurs Content Types (partagés)

```dataview
TABLE
  file.name as "Paragraph",
  length(file.inlinks) as "Utilisé par N entités",
  file.inlinks as "Entités"
FROM "Entities/Paragraphs"
WHERE type = "paragraph"
  AND length(file.inlinks) > 1
SORT length(file.inlinks) DESC
```

---

## Dataview JS — Requêtes Complexes

Pour des analyses qui nécessitent du calcul, utiliser `dataviewjs` :

```javascript
// Générer un rapport d'architecture complet
const entities = dv.pages('"Entities"')
  .where(p => p.type === "content_type" || p.type === "paragraph");

const fields = dv.pages('"Fields"')
  .where(p => p.type === "field_storage");

const orphanFields = fields.where(f => f.file.inlinks.length === 0);
const missingTemplates = entities.where(e => 
  e.file.outlinks.filter(l => l.path.includes("Theme/Templates")).length === 0
);

dv.header(2, "Rapport d'Architecture");
dv.paragraph(`**${entities.length}** entités | **${fields.length}** champs | **${orphanFields.length}** champs orphelins | **${missingTemplates.length}** entités sans template`);

dv.header(3, "Champs Orphelins");
dv.list(orphanFields.map(f => f.file.link));

dv.header(3, "Entités Sans Template Twig");
dv.table(
  ["Entité", "Type", "Créée le"],
  missingTemplates.map(e => [e.file.link, e.type, e.created])
);
```

---

## Requêtes de Santé du Vault

### 21. Paragraphs sans template Twig associé (version correcte)

La requête vérifie si une note `#twig_template` contient un **backlink** vers le paragraph — c'est la relation sémantique correcte.

```dataviewjs
// Paragraphs sans template Twig associé
dv.table(["Paragraph", "Bundle", "Champs"],
  dv.pages('#paragraph')
    .where(p => !dv.pages('#twig_template')
      .some(t => t.file.outlinks.some(l => l.path === p.file.path)))
    .map(p => [p.file.link, p.bundle, p.fields?.length ?? 0])
)
```

### 22. Migrations non documentées dans le vault

Migrations déclarées dans la config Drupal (notes de type `migration`) mais sans note de documentation dans le vault.

```dataviewjs
// Migrations sans documentation associée
dv.table(
  ["Migration ID", "Source", "Destination", "Tags"],
  dv.pages('#migration')
    .where(m => !m.documented || m.documented === false)
    .map(m => [m.file.link, m.source_plugin ?? "—", m.destination_plugin ?? "—", m.tags ?? ""])
)
```

### 23. Notes de type twig_template sans entity/bundle référencé

Templates Twig documentés mais qui ne référencent aucune entité Drupal (ni content type ni paragraph).

```dataviewjs
// Templates Twig "orphelins" — sans lien vers une entité
dv.table(
  ["Template", "entity_type", "bundle"],
  dv.pages('#twig_template')
    .where(t => {
      const linksToEntities = t.file.outlinks.some(l =>
        l.path.includes("Entities/") || l.path.includes("Paragraphs/")
      );
      return !linksToEntities;
    })
    .map(t => [t.file.link, t.entity_type ?? "—", t.bundle ?? "—"])
)
```

### 24. Résumé santé du vault

```dataviewjs
// Dashboard de santé global
const paragraphs = dv.pages('#paragraph');
const templates  = dv.pages('#twig_template');
const migrations = dv.pages('#migration');
const fields     = dv.pages('#field_storage');

const paragraphsWithTemplate = paragraphs.filter(p =>
  templates.some(t => t.file.outlinks.some(l => l.path === p.file.path))
);
const orphanFields = fields.filter(f => f.file.inlinks.length === 0);

dv.header(3, "Santé du vault");
dv.paragraph(`
| Métrique | Valeur |
|----------|--------|
| Paragraphs couverts par un template | **${paragraphsWithTemplate.length}** / ${paragraphs.length} |
| Champs orphelins (non référencés) | **${orphanFields.length}** |
| Migrations documentées | **${migrations.length}** |
| Templates Twig documentés | **${templates.length}** |
`);
```

---

## Notes de Migration — Frontmatter standard

Utiliser ce frontmatter pour documenter les migrations Drupal dans le vault :

```yaml
---
type: migration
migration_id: mon_module_articles
source_plugin: d7_node:article
destination_plugin: entity:node
bundle: article
tags:
  - migration
  - content
status: active      # active | deprecated | draft
documented: true
dependencies:
  - mon_module_users
created: 2025-01-15
updated: 2025-03-10
---
```

**Champs requis :**

| Champ | Description | Exemple |
|-------|-------------|---------|
| `type` | Toujours `migration` | `migration` |
| `migration_id` | ID machine de la migration | `mon_module_articles` |
| `source_plugin` | Plugin source Drupal | `d7_node:article` |
| `destination_plugin` | Plugin destination | `entity:node` |
| `bundle` | Bundle cible | `article` |
| `status` | État de la migration | `active` |
| `documented` | Booléen — permet les requêtes de santé | `true` |
| `dependencies` | Migrations qui doivent tourner avant | `[mon_module_users]` |
