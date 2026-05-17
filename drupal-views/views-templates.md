---
name: drupal-views — templates Twig
description: Tous les templates Views, leur hiérarchie de suggestions, les variables disponibles, et les patterns d'override pour les affichages de liste, champs, lignes, et vides.
---

# Views Templates Twig — Référence Complète

## ⚠️ Règle Absolue — Nommage des Fichiers

Les templates Views utilisent des **tirets**, les commentaires Twig debug montrent des **underscores** (nom du hook) — toujours convertir `__` → `--` dans les noms de fichiers.

```
# Commentaire debug (hook name) :      views-view__MON-VIEW.html.twig
# Nom du fichier réel (tirets) :        views-view--mon-view.html.twig
```

---

## Hiérarchie des Suggestions — Du Général au Spécifique

```
# Container principal de la View
views-view.html.twig                                          (le plus général)
views-view--VIEW-ID.html.twig                                 (par View)
views-view--VIEW-ID--DISPLAY-ID.html.twig                     (par display)

# Style (grid, list, table, etc.)
views-view-unformatted.html.twig
views-view-unformatted--VIEW-ID.html.twig
views-view-unformatted--VIEW-ID--DISPLAY-ID.html.twig

# Une ligne de résultat (row)
views-view-fields.html.twig                                   (ligne = ensemble de champs)
views-view-fields--VIEW-ID.html.twig
views-view-fields--VIEW-ID--DISPLAY-ID.html.twig

# Un champ spécifique dans une ligne
views-view-field.html.twig
views-view-field--FIELD-ID.html.twig
views-view-field--FIELD-ID--VIEW-ID.html.twig
views-view-field--FIELD-ID--VIEW-ID--DISPLAY-ID.html.twig     (le plus spécifique)
```

**Pour activer Twig Debug :** `/admin/config/development/performance` + `Twig debug = ON` ou `services.yml`.

---

## Template Container Principal — `views-view.html.twig`

```twig
{# web/themes/custom/mon_theme/templates/views/views-view--commandes-liste--page-1.html.twig #}

{%
  set classes = [
    dom_id ? 'js-view-dom-id-' ~ dom_id,
    'view',
    'view-' ~ id|clean_class,
    'view-display-' ~ display_id,
  ]
%}

<div{{ attributes.addClass(classes) }}>
  {% if header %}
    <div class="view-header">
      {{ header }}
    </div>
  {% endif %}

  {% if exposed %}
    <div class="view-filters">
      {{ exposed }}
    </div>
  {% endif %}

  {% if attachment_before %}
    <div class="attachment attachment-before">
      {{ attachment_before }}
    </div>
  {% endif %}

  {% if rows %}
    <div class="view-content">
      {{ rows }}
    </div>
  {% elseif empty %}
    <div class="view-empty">
      {{ empty }}
    </div>
  {% endif %}

  {% if pager %}
    <div class="view-pager">
      {{ pager }}
    </div>
  {% endif %}

  {% if footer %}
    <div class="view-footer">
      {{ footer }}
    </div>
  {% endif %}
</div>

{# Variables disponibles :
   id              : machine name de la View
   display_id      : ID du display actuel
   dom_id          : ID DOM unique (pour AJAX)
   attributes      : objet attributes HTML
   header          : render array du header
   footer          : render array du footer
   exposed         : render array du formulaire exposé
   rows            : render array des résultats
   empty           : render array si aucun résultat
   pager           : render array de la pagination
   attachment_before / attachment_after : views attachées
   more            : lien "Voir plus" (si configuré)
#}
```

---

## Template Ligne — `views-view-fields.html.twig`

```twig
{# Template pour l'ensemble des champs d'une ligne de résultat #}
{# views-view-fields--commandes-liste--page-1.html.twig #}

<article class="commande-card commande-card--{{ fields.statut.content|clean_class }}">

  <header class="commande-card__header">
    <span class="commande-card__reference">{{ fields.reference.content }}</span>
    <span class="commande-card__statut">{{ fields.statut.content }}</span>
  </header>

  <div class="commande-card__body">
    {% if fields.montant.content %}
      <p class="commande-card__montant">{{ fields.montant.content }}</p>
    {% endif %}

    {% if fields.created.content %}
      <time class="commande-card__date">{{ fields.created.content }}</time>
    {% endif %}
  </div>

</article>

{# Variables disponibles dans views-view-fields :
   fields            : tableau indexé par field_id
     fields.MON_CHAMP.content    : valeur rendue du champ (HTML safe)
     fields.MON_CHAMP.raw        : valeur brute (PAS échappée)
     fields.MON_CHAMP.label      : label du champ
     fields.MON_CHAMP.label_html : label avec balise <span> (si affiché)
     fields.MON_CHAMP.wrapper_suffix / wrapper_prefix : wrappers
     fields.MON_CHAMP.separator  : séparateur entre valeurs multi
   row               : objet ResultRow (accès aux données brutes)
#}
```

---

## Template Champ Individuel — `views-view-field.html.twig`

```twig
{# views-view-field--montant--commandes-liste.html.twig #}
{# Surcharge l'affichage d'UN seul champ dans toutes les lignes #}

<span class="commande-montant commande-montant--{{ output|clean_class }}">
  {{ output }}
</span>

{# Variables disponibles :
   output  : valeur rendue du champ (MarkupInterface)
   field   : l'objet FieldPluginBase
   row     : l'objet ResultRow
   view    : l'objet ViewExecutable
#}
```

---

## Template Style Non-Formaté — `views-view-unformatted.html.twig`

```twig
{# views-view-unformatted--commandes-liste--page-1.html.twig #}
{# Container du style (liste de toutes les lignes) #}

<div class="commandes-grid">
  {% for row in rows %}
    <div{{ row.attributes.addClass('commandes-grid__item') }}>
      {{- row.content -}}
    </div>
  {% endfor %}
</div>

{# Variables :
   rows[].content    : render array de la ligne (template views-view-fields)
   rows[].attributes : attributs HTML de la ligne (Attribute object)
   title             : titre de la View (si configuré)
   view              : ViewExecutable
#}
```

---

## Template Table — `views-view-table.html.twig`

```twig
{# Surcharger le style "Table" de Views #}
{# views-view-table--commandes-liste--page-1.html.twig #}

<div class="table-responsive">
  <table{{ attributes }}>
    {% if caption_needed %}
      <caption>
        {% if caption %}
          {{ caption }}
        {% else %}
          {{ title }}
        {% endif %}
        {% if (summary_element is not empty) %}
          {{ summary_element }}
        {% endif %}
      </caption>
    {% endif %}

    <thead>
      <tr>
        {% for key, column in header %}
          <th{{ column.attributes }}>
            {{ column.content }}
          </th>
        {% endfor %}
      </tr>
    </thead>

    <tbody>
      {% for key, row in rows %}
        <tr{{ row.attributes.addClass(loop.index is odd ? 'odd' : 'even') }}>
          {% for key, column in row.columns %}
            <td{{ column.attributes }}>
              {{ column.content }}
            </td>
          {% endfor %}
        </tr>
      {% endfor %}
    </tbody>
  </table>
</div>
```

---

## Template Vide — `views-view-fields.html.twig` avec empty

```twig
{# Afficher un message custom quand il n'y a aucun résultat #}
{# Configurable dans Views UI : No Results Behavior → Global: Text area #}
{# Pour surcharger le rendu du message vide, utiliser le hook preprocess : #}

{# Dans mon_theme.theme : #}
{#
function mon_theme_preprocess_views_view(&$variables) {
  $view = $variables['view'];
  if ($view->id() === 'commandes_liste' && empty($view->result)) {
    $variables['empty_message'] = t('Vous n\'avez pas encore de commandes.');
  }
}
#}
```

---

## Ajouter des Suggestions Depuis PHP

```php
// Dans mon_theme.theme — hook_theme_suggestions_views_view_alter()
function mon_theme_theme_suggestions_views_view_alter(array &$suggestions, array $variables): void {
  $view = $variables['view'];

  // Ajouter une suggestion selon le bundle du nœud affiché
  if ($view->id() === 'contenu_liste') {
    $bundle = $view->args[0] ?? NULL;
    if ($bundle) {
      $suggestions[] = 'views_view__contenu_liste__' . $bundle;
    }
  }
}

// Pour les lignes :
function mon_theme_theme_suggestions_views_view_fields_alter(array &$suggestions, array $variables): void {
  $view = $variables['view'];
  $row = $variables['row'];

  // Suggestion différente selon le statut de la commande
  $statut = $row->mon_module_commandes_statut ?? NULL;
  if ($view->id() === 'commandes_liste' && $statut) {
    $suggestions[] = 'views_view_fields__commandes_liste__' . $statut;
    // → views-view-fields--commandes-liste--confirmed.html.twig
  }
}
```

---

## Récapitulatif des Variables par Template

| Template | Variables clés |
|---------|---------------|
| `views-view` | `id`, `display_id`, `rows`, `header`, `footer`, `exposed`, `pager`, `empty`, `dom_id` |
| `views-view-unformatted` | `rows[].content`, `rows[].attributes`, `title`, `view` |
| `views-view-table` | `header[]`, `rows[].columns[]`, `caption`, `attributes` |
| `views-view-fields` | `fields[ID].content`, `fields[ID].label`, `row`, `view` |
| `views-view-field` | `output`, `field`, `row`, `view` |
| `views-view-grid` | `items[]`, `attributes`, `num_columns`, `alignment` |
