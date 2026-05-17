---
name: drupal-search — facets
description: Configuration des facettes avec le module Facets pour Drupal - types de facettes, widgets, processeurs, compteurs, liens, intégration Views et Search API.
---

# Facets — Configuration & Personnalisation

## Installation et Prérequis

```bash
# Prérequis : Search API + un index configuré
composer require drupal/facets
drush en facets -y

# Facets Summary (résumé des filtres actifs)
composer require drupal/facets_summary
drush en facets_summary -y
```

**Prérequis :** avoir un index Search API configuré avec les champs sur lesquels créer des facettes.

---

## Créer une Facette

```yaml
# config/install/facets.facet.tags.yml
langcode: fr
status: true
id: tags
name: 'Thématiques'
url_alias: tags
weight: 0
show_only_one_result: false
field_identifier: field_tags_name      # Champ indexé dans Search API
facet_source_id: 'search_api:views_page__recherche__page_1'  # Source
# Format : search_api:views_{display_type}__{view_id}__{display_id}
facet_source_config: {}
widget:
  type: facets_checkbox_widget          # Type de widget
  config:
    show_numbers: true                  # Afficher le compteur (52)
    soft_limit: 10                      # Masquer après 10 valeurs
    soft_limit_settings:
      show_less_label: 'Voir moins'
      show_more_label: 'Voir plus'
    infinite_scroll: false
processor_configs:
  count_limit:                          # Masquer les valeurs avec 0 résultats
    processor_id: count_limit
    weights:
      build: -10
    settings:
      minimum_count: 1
  display_value_widget_order:           # Trier par nom de valeur
    processor_id: display_value_widget_order
    weights:
      sort: -10
    settings:
      sort: ASC
  translate_entity:                     # Traduire les labels (taxonomy terms)
    processor_id: translate_entity
    weights:
      build: -5
    settings: {}
empty_behavior:
  behavior: none                        # 'none' | 'text'
cache:
  query_cacheability: 1
```

---

## Widgets Disponibles — Référence

| Widget | ID | Usage |
|--------|-----|-------|
| Checkboxes | `facets_checkbox_widget` | Sélection multiple (recommandé) |
| Liens | `facets_links_widget` | Chaque valeur = un lien |
| Lien unique | `links_widget` | Un clic = filtre exclusif |
| Select | `select_widget` | Dropdown HTML `<select>` |
| Range | `range_widget` | Filtrage par plage numérique |
| Dropdown | `dropdown` | Select stylisé |
| Hidden | `facets_hidden_widget` | Facette invisible (pour filtrage programmatique) |

---

## Processeurs — Configuration

### Les Processeurs Essentiels

```yaml
processor_configs:
  # Masquer les valeurs à 0 résultats (TOUJOURS activer)
  count_limit:
    processor_id: count_limit
    settings:
      minimum_count: 1      # Valeur minimale : 1 résultat

  # Trier par nombre de résultats (décroissant)
  count_widget_order:
    processor_id: count_widget_order
    settings:
      sort: DESC            # ASC ou DESC

  # Trier alphabétiquement
  display_value_widget_order:
    processor_id: display_value_widget_order
    settings:
      sort: ASC

  # Exclure des valeurs spécifiques de la facette
  exclude_specified_items:
    processor_id: exclude_specified_items
    settings:
      exclude: "Brouillon\nArchivé"    # Une valeur par ligne

  # Traduction des labels (taxonomy terms traduits)
  translate_entity:
    processor_id: translate_entity

  # Transformer les IDs en labels lisibles
  list_item:
    processor_id: list_item   # Pour les champs List (select)

  # URL processing — activer si facettes dans l'URL
  url_processor_handler:
    processor_id: url_processor_handler
```

---

## Placer les Facettes en Région ou Bloc

```yaml
# Placer une facette comme bloc dans une région
# config/install/block.block.facet_tags.yml
langcode: fr
status: true
id: facet_tags
theme: mon_theme
region: sidebar_first
weight: 0
plugin: facet_block:tags     # 'facet_block:' + ID de la facette
settings:
  label: 'Thématiques'
  label_display: visible
visibility:
  request_path:
    id: request_path
    negate: false
    pages: "/recherche\n/recherche/*"   # Afficher seulement sur /recherche
```

---

## Template Twig pour Facettes

```twig
{# templates/facets/facets-item-list.html.twig #}
{# Surcharger le template de liste de facettes #}

{% if items %}
  <ul class="facet-list">
    {% for item in items %}
      <li class="facet-list__item {{ item.value|clean_class }}">
        <label class="facet-list__label{% if item.values.active %} facet-list__label--active{% endif %}">
          <input
            type="checkbox"
            class="facet-list__checkbox"
            {% if item.values.active %}checked{% endif %}
            onchange="this.form.submit()"
          >
          <span class="facet-list__text">{{ item.value }}</span>
          {% if show_numbers %}
            <span class="facet-list__count">({{ item.values.count }})</span>
          {% endif %}
        </label>
        {# Lien pour désactiver le filtre #}
        {% if item.values.active %}
          <a href="{{ item.values.url }}" class="facet-list__remove" aria-label="Supprimer le filtre {{ item.value }}">×</a>
        {% else %}
          <a href="{{ item.values.url }}" class="facet-list__link">{{ item.value }}</a>
        {% endif %}
      </li>
    {% endfor %}
  </ul>
{% endif %}
```

---

## Facets Summary — Résumé des Filtres Actifs

```yaml
# config/install/facets_summary.facets_summary.resume_filtres.yml
langcode: fr
status: true
id: resume_filtres
name: 'Filtres actifs'
facet_source_id: 'search_api:views_page__recherche__page_1'
facets:
  tags:
    checked: true            # Inclure cette facette dans le résumé
    label: 'Thématique : '
    separator: ', '
    show_count: false
  type:
    checked: true
    label: 'Type : '
    separator: ', '
    show_count: false
processor_configs:
  reset_facets:
    processor_id: reset_facets
    settings:
      link_text: 'Réinitialiser tous les filtres'
```

---

## Facettes Programmatiques (API PHP)

```php
// Accéder aux facettes dans un Controller ou Service
use Drupal\facets\FacetManager\DefaultFacetManager;

class MaRechercheController extends ControllerBase {

  public function __construct(
    private readonly DefaultFacetManager $facetManager,
  ) {}

  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('facets.manager'),
    );
  }

  public function recherche(): array {
    // Obtenir les facettes pour une source
    $facets = $this->facetManager->getFacetsByFacetSourceId(
      'search_api:views_page__recherche__page_1'
    );

    // Processer les facettes (calcul des valeurs + compteurs)
    $this->facetManager->updateResults(
      'search_api:views_page__recherche__page_1'
    );

    $build = [];
    foreach ($facets as $facet) {
      $build['facet_' . $facet->id()] = $this->facetManager->build($facet);
    }

    return $build;
  }
}
```

---

## Debug des Facettes

```bash
# Vérifier que la facette est bien liée à la bonne source
drush php:eval "
\$facets = \Drupal::entityTypeManager()->getStorage('facets_facet')->loadMultiple();
foreach (\$facets as \$facet) {
  echo \$facet->id() . ': source=' . \$facet->getFacetSourceId() . PHP_EOL;
}
"

# Vérifier que le champ est bien indexé
drush php:eval "
\$index = \Drupal::entityTypeManager()->getStorage('search_api_index')->load('articles_index');
\$fields = \$index->getFields();
foreach (\$fields as \$key => \$field) {
  echo \$key . ': ' . \$field->getType() . PHP_EOL;
}
"

# Vider le cache après config facette
drush cr
```

---

## Checklist Post-Configuration

```
[ ] L'index Search API a été réindexé après ajout des champs de facettes
    drush sapi-c articles_index && drush sapi-i articles_index

[ ] La facette est liée à la bonne source Views
    (search_api:views_{type}__{view_id}__{display_id})

[ ] Le processeur count_limit (min=1) est activé
    → Évite les facettes avec 0 résultats

[ ] Les blocs de facettes sont placés dans la bonne région
    /admin/structure/block

[ ] Le cache a été vidé
    drush cr

[ ] La Vue est en mode AJAX pour que les facettes rechargent sans refresh
    Views → Advanced → Use AJAX → Yes
```
