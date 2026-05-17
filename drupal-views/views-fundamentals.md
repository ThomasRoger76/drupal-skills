---
name: drupal-views — fundamentals
description: Configuration UI complète de Drupal Views - types d'affichage (Page, Block, REST, Feed), champs, filtres, tris, relations, pagination, accès, en-têtes, pied de pages, et views attachées.
---

# Views UI — Configuration Complète

## Types de Display

| Display | Usage | URL |
|---------|-------|-----|
| **Page** | Page Drupal avec URL propre | `/mes-articles` |
| **Block** | Bloc placé dans une région | Géré via /admin/structure/block |
| **Embed** | Intégré dans un Controller PHP | Via `Views::getView()->render()` |
| **REST Export** | JSON/XML/CSV endpoint | `/api/articles?_format=json` |
| **Feed (RSS)** | Flux RSS | `/articles/feed` |
| **Attachment** | Attaché avant/après une autre display | Hérite des filtres exposés du parent |

---

## Ajouter et Configurer des Champs

```
Views UI → Fields → Add

Champs disponibles selon la table de base :
├── Champs d'entités (node.title, user.name...)
├── Champs Entity Reference (avec relation)
├── Champs globaux (Global: Custom text, Global: View, Global: Null)
├── Champs calculés (Results counter, Nothing...)
└── Champs custom (via hook_views_data)
```

### Options de champ importantes

| Option | Usage |
|--------|-------|
| Label | Libellé affiché (vide = pas de label) |
| Formatter | Mise en forme (Image style, Date format...) |
| Rewrite results | Remplacer la valeur par du HTML custom (tokens disponibles) |
| Exclude from display | Calculer sans afficher (pour Rewrite) |
| Link to entity | Rendre le champ cliquable vers l'entité |
| Empty text | Texte si la valeur est vide |

---

## Filtres

### Types de filtres

| Type | Opérateurs disponibles | Usage |
|------|----------------------|-------|
| String | Contient, =, Commence par, IN | Textes |
| Numeric | =, !=, <, >, BETWEEN | Nombres |
| Boolean | Est vrai / Est faux | Champs yes/no |
| Date | Date = / < / > / BETWEEN, "relative" | Dates |
| In Operator | IN, NOT IN | Listes de valeurs |
| Bundle | Par type d'entité | Content types |
| Language | Par langcode | Multilingue |

### Filtres exposés

Un filtre peut être "exposé" — l'utilisateur peut le modifier via un formulaire affiché dans la View.

```
Filter → Expose this filter to visitors
  ├── Label : texte affiché à côté du filtre
  ├── Identifier : clé utilisée dans l'URL (?title=foo)
  ├── Required : l'utilisateur DOIT remplir ce filtre
  ├── Remember : mémoriser la valeur (cookie)
  └── Multiple : autoriser la sélection multiple (IN)
```

---

## Tri

```
Sort criteria → Add

Options :
├── Ascending (ASC) / Descending (DESC)
├── Expose as sortable (allow user to re-sort)
└── Override URL (changer le paramètre URL de tri)
```

**Ordre des tris :** drag & drop — le premier tri prime en cas d'égalité.

---

## Relations

Les relations permettent de JOIN une table additionnelle — accéder aux champs d'entités référencées.

```
Relationships → Add

Exemple : Node → field_author → Users (join sur uid)
  → permet d'ajouter des champs "Users: Name", "Users: Email"
  → la relation est disponible dans tous les champs, filtres, tris
```

**Relation avec "Required"** : INNER JOIN (exclut les nœuds sans auteur)
**Relation sans "Required"** : LEFT JOIN (inclut tous les nœuds)

---

## Contextual Filters (Arguments)

```
Advanced → Contextual filters → Add

Configuration d'un argument :
  ├── "When the filter value is NOT available"
  │     ├── Display all results (pas de filtre)
  │     ├── Hide view (aucun affichage)
  │     ├── Display empty text
  │     └── Provide default value
  │           ├── Fixed value
  │           ├── PHP Code
  │           ├── Current user's UID
  │           └── URL component
  │
  ├── "When the filter value IS available or a default is provided"
  │     ├── Override title (ex: "Articles de %1")
  │     └── Specify validation criteria
  │           ├── Entity : node (valide que l'argument est un NID)
  │           └── fail: "not found" (404 si invalide)
  │
  └── Glossary mode (regrouper par première lettre)
```

**URL des contextual filters :**
- `1 argument` : `/vue/valeur`
- `2 arguments` : `/vue/valeur1/valeur2`
- `valeur optionnelle` : configurer "ALL" comme exception

---

## Pagination

```
Advanced → Pager

Types :
├── Display all items (pas de pagination)
├── Full (pager Drupal avec numéros de page)
├── Mini (Précédent / Suivant uniquement)
└── Some items (N items fixes, pas de pager visible)

Options Full pager :
├── Items per page : nombre d'items par page
├── Expose items per page : laisser l'utilisateur choisir
└── Offset : sauter les N premiers résultats
```

---

## Access — Contrôle d'Accès

```
Advanced → Access → Change

Types d'accès :
├── None (❌ DÉCONSEILLÉ pour données sensibles)
├── Permission (Drupal permission system)
│     → sélectionner une permission Drupal
├── Role
│     → sélectionner un ou plusieurs rôles
└── Custom Access Plugin
      → plugin PersonalisedAccessPlugin (voir custom-handlers.md)
```

---

## Header / Footer / Empty Text

```
Advanced → Header / Footer / No results behavior

Types disponibles :
├── Global: Text area (HTML + tokens)
│     → tokens disponibles : [view:total-rows], [view:title]...
├── Global: Unfiltered text (HTML sans échappement)
├── Global: View (afficher une autre View)
├── Global: Result summary
│     → ex: "Affichage des éléments 1 à 10 sur 42"
└── Custom area handler (voir custom-handlers.md)
```

---

## Views Attachées (Attachments)

```
Display → Add → Attachment

Configuration :
├── "Attach to" : quels displays afficher avec
├── "Attachment position" : Before / After / Both
├── "Inherit exposed filters" : YES → partage les filtres exposés du parent
└── "Inherit pager" : utiliser la pagination du parent
```

**Usage typique :** View principale = liste, Attachment avant = statistiques/résumé.

---

## Cache — Configuration par Display

```
Advanced → Caching

Options :
├── None (pas de cache)
├── Time-based caching
│     → Cache query results : durée en secondes
│     └── Cache rendered output : durée en secondes
└── Tag-based caching (recommandé)
      → cache invalide automatiquement quand les entités changent
```

**Recommandation :** Tag-based pour le contenu qui change. Time-based pour les données rarement modifiées.

---

## Other Settings — Paramètres Avancés

| Setting | Usage |
|---------|-------|
| Machine name | Identifiant interne du display |
| Administrative comment | Notes pour les développeurs |
| Use AJAX | AJAX pour les filtres exposés et la pagination |
| Hide attachments in summary | Masquer les attachments dans les vues résumées |
| Contextual links | Afficher les liens d'édition rapide |
| Use aggregation | GROUP BY (agréger les résultats) |

---

## Exporter / Importer une View (Config)

```bash
# Exporter la config d'une View
drush cex -y
# → config/sync/views.view.MA_VIEW.yml

# Importer
drush cim -y

# Voir la config d'une View sans l'exporter
drush config:get views.view.ma_view

# Créer un override d'une View via drush
drush config:set views.view.ma_view display.default.display_options.use_ajax '1'
```
