---
name: drupal-views — AJAX et exposed filters
description: Configuration des exposed filters AJAX dans Views, Better Exposed Filters (BEF), filtres dépendants, et gestion de la soumission automatique.
---

# Views AJAX & Exposed Filters — Référence Complète

## Activer l'AJAX sur une View

```yaml
# Dans la configuration de la View :
display_options:
  use_ajax: true       # Active l'AJAX pour le rechargement des résultats
  exposed_form:
    type: basic
    options:
      submit_button: 'Filtrer'
      reset_button: true
      reset_button_label: 'Réinitialiser'
      exposed_sorts_label: 'Trier par'
      expose_sort_order: true
      sort_asc_label: 'Croissant'
      sort_desc_label: 'Décroissant'
```

**Via l'UI :** Views → Advanced → Use AJAX → Yes

---

## Exposed Filters — Configuration Correcte

```yaml
# Filtre texte exposé (recherche sur le titre)
display_options:
  filters:
    title:
      id: title
      table: node_field_data
      field: title
      relationship: none
      group_type: group
      admin_label: Titre
      plugin_id: string
      operator: CONTAINS
      value: ''
      exposed: true
      expose:
        operator_id: title_op
        label: 'Rechercher par titre'
        description: ''
        use_operator: false
        operator: title_op
        operator_limit_selection: false
        operator_list: []
        identifier: title
        required: false
        remember: false
        multiple: false
        placeholder: 'Saisir un terme...'
```

---

## Better Exposed Filters (BEF)

BEF améliore l'UI des filtres exposés : checkboxes, boutons radio, sliders, sélecteurs de date, etc.

```bash
composer require drupal/better_exposed_filters
drush en better_exposed_filters -y
```

### Configuration via l'UI

1. Views → Exposed Form → Exposed form style → Better Exposed Filters
2. Par filtre → modifier → "More" → Exposed settings → Plugin BEF

### Widgets BEF disponibles

| Widget | Usage |
|--------|-------|
| `default` | Widget Drupal standard (select) |
| `bef_checkboxes` | Checkboxes (choix multiples) |
| `bef_radios` | Boutons radio |
| `bef_links` | Liens cliquables |
| `bef_hidden` | Champ caché (valeur fixe) |
| `bef_slider` | Slider numérique (nécessite jQuery UI) |
| `search_api_autocomplete` | Autocomplete (Search API) |

---

## Auto-Submit — Soumettre sans Bouton

```php
// Activer la soumission automatique quand un filtre change
// Via hook_form_views_exposed_form_alter()
function mon_module_form_views_exposed_form_alter(array &$form, FormStateInterface $form_state): void {
  $view = $form_state->get('view');

  if ($view->id() !== 'commandes_liste') {
    return;
  }

  // Auto-submit : soumettre le formulaire dès qu'un select change
  $form['#attached']['library'][] = 'core/drupal';
  $form['type']['#attributes']['data-drupal-autosubmit'] = TRUE;
  $form['statut']['#attributes']['data-drupal-autosubmit'] = TRUE;

  // Ou via Better Exposed Filters — cocher "Auto-submit" dans la config du filtre
}
```

---

## hook_views_exposed_form_alter — Personnaliser les Filtres

```php
/**
 * Implements hook_form_views_exposed_form_alter().
 * Personnaliser le formulaire de filtres exposés d'une View.
 */
function mon_module_form_views_exposed_form_alter(array &$form, FormStateInterface $form_state): void {
  $view = $form_state->get('view');

  if ($view->id() !== 'recherche_articles') {
    return;
  }

  // Modifier le placeholder d'un champ de recherche
  if (isset($form['title'])) {
    $form['title']['#attributes']['placeholder'] = t('Rechercher un article...');
    $form['title']['#attributes']['class'][] = 'search-input--principal';
  }

  // Ajouter une valeur par défaut selon l'utilisateur connecté
  if (isset($form['auteur']) && \Drupal::currentUser()->isAuthenticated()) {
    $form['auteur']['#default_value'] = \Drupal::currentUser()->id();
  }

  // Grouper des filtres dans un fieldset collapsible
  $form['filtres_avances'] = [
    '#type' => 'details',
    '#title' => t('Filtres avancés'),
    '#open' => FALSE,
  ];

  if (isset($form['date_debut'])) {
    $form['filtres_avances']['date_debut'] = $form['date_debut'];
    unset($form['date_debut']);
  }

  if (isset($form['date_fin'])) {
    $form['filtres_avances']['date_fin'] = $form['date_fin'];
    unset($form['date_fin']);
  }

  // Modifier l'action du formulaire pour AJAX
  $form['#action'] = '/recherche';

  // Ajouter validation custom
  $form['#validate'][] = 'mon_module_recherche_validate';
}

function mon_module_recherche_validate(array $form, FormStateInterface $form_state): void {
  $min = $form_state->getValue('montant_min');
  $max = $form_state->getValue('montant_max');

  if ($min && $max && $min > $max) {
    $form_state->setError(
      $form['montant_max'],
      t('Le montant maximum doit être supérieur au montant minimum.')
    );
  }
}
```

---

## Contextual Filters — Filtrage par Argument URL

```yaml
# Configurer un contextual filter sur la View (arguments dans l'URL)
display_options:
  arguments:
    nid:
      id: nid
      table: node_field_data
      field: nid
      relationship: none
      plugin_id: node_nid
      default_action: default   # Que faire si l'argument est absent ?
      default_argument_type: fixed
      default_argument_options:
        argument: '0'           # Valeur par défaut si absent
      exception:
        value: all
        title_enable: false
      title_enable: true
      title: 'Articles de %1'   # %1 = valeur de l'argument
      summary_options:
        base_path: ''
        count: true
        override: false
        items_per_page: 25
      validate:
        type: 'entity:node'     # Valider que l'argument est un NID valide
        fail: 'not found'       # 404 si invalide
      validate_options:
        bundles:
          article: article
```

**Via PHP :** passer les arguments programmatiquement :
```php
$view = Views::getView('articles_par_auteur');
$view->setDisplay('block_1');
$view->setArguments([\Drupal::currentUser()->id()]);  // UID comme argument
$view->execute();
```

---

## AJAX Custom — Remplacer le Comportement AJAX Views

```javascript
// Surcharger l'AJAX Views pour ajouter un loading indicator
(function (Drupal, once) {
  'use strict';

  // Intercepter le début d'une requête AJAX Views
  $(document).on('ajaxStart', function () {
    once('views-ajax-loading', '.view').forEach(function (el) {
      el.classList.add('view--loading');
    });
  });

  // Intercepter la fin d'une requête AJAX Views
  $(document).on('views_ajax_success', function (event, view) {
    document.querySelectorAll('.view--loading').forEach(function (el) {
      el.classList.remove('view--loading');
    });
  });

})(Drupal, once);
```

```php
// Pré-remplir les filtres exposés depuis une requête externe
function mon_module_views_pre_build(\Drupal\views\ViewExecutable $view): void {
  if ($view->id() !== 'recherche' || $view->current_display !== 'page_1') {
    return;
  }

  // Si un paramètre URL externe doit pré-remplir un filtre
  $request = \Drupal::request();
  $q = $request->query->get('q');

  if ($q) {
    $exposed = $view->getExposedInput();
    $exposed['title'] = $q;
    $view->setExposedInput($exposed);
  }
}
```
