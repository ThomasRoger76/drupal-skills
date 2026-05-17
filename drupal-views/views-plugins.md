---
name: drupal-views — plugins style et display
description: Créer des plugins Views custom - Style plugins (grille, slider, carte), Display plugins (iCal, Sitemap, export custom), et Row plugins. Avec les attributs PHP D11 et les patterns corrects.
---

# Views Style & Display Plugins Custom

## Hiérarchie des Plugins Views

```
Display Plugin   → Détermine COMMENT la View est accessible (page, block, REST...)
  ↓
Style Plugin     → Détermine l'ORGANISATION des résultats (liste, grille, table...)
  ↓
Row Plugin       → Détermine le RENDU de chaque ligne individuelle
  ↓
Field Plugins    → Rendu de chaque champ dans une ligne
```

---

## Style Plugin Custom

```php
<?php
// src/Plugin/views/style/CarteStyle.php
namespace Drupal\mon_module\Plugin\views\style;

use Drupal\Core\Form\FormStateInterface;
use Drupal\views\Plugin\views\style\StylePluginBase;

/**
 * Style Views "Carte" — affiche les résultats en grille de cartes.
 *
 * @ViewsStyle(
 *   id = "mon_module_carte",
 *   title = @Translation("Carte (grille)"),
 *   help = @Translation("Affiche les résultats en grille de cartes."),
 *   theme = "views_view_carte",
 *   display_types = {"normal"}
 * )
 */
// D11 : #[ViewsStyle(...)] attribute
class CarteStyle extends StylePluginBase {

  /**
   * Autoriser les champs Fields dans ce style.
   */
  protected $usesFields = TRUE;

  /**
   * Autoriser les options de regroupement.
   */
  protected $usesGrouping = FALSE;

  /**
   * Options du style.
   */
  protected function defineOptions(): array {
    $options = parent::defineOptions();
    $options['colonnes'] = ['default' => 3];
    $options['image_style'] = ['default' => 'thumbnail'];
    return $options;
  }

  /**
   * Formulaire de configuration du style.
   */
  public function buildOptionsForm(&$form, FormStateInterface $form_state): void {
    parent::buildOptionsForm($form, $form_state);

    $form['colonnes'] = [
      '#type' => 'select',
      '#title' => $this->t('Nombre de colonnes'),
      '#options' => [2 => '2', 3 => '3', 4 => '4', 6 => '6'],
      '#default_value' => $this->options['colonnes'],
    ];

    $form['image_style'] = [
      '#type' => 'select',
      '#title' => $this->t('Style d\'image'),
      '#options' => image_style_options(),
      '#default_value' => $this->options['image_style'],
    ];
  }

  /**
   * Rendu du style — appelé une seule fois pour toutes les lignes.
   */
  public function render(): array {
    $groups = $this->renderGrouping($this->view->result, $this->options['grouping']);

    return [
      '#theme' => 'views_view_carte',
      '#view' => $this->view,
      '#rows' => $groups,
      '#options' => $this->options,
      '#colonnes' => (int) $this->options['colonnes'],
    ];
  }
}
```

**Template associé :**
```php
// mon_module.module — déclarer le thème
function mon_module_theme(): array {
  return [
    'views_view_carte' => [
      'variables' => [
        'view' => NULL,
        'rows' => [],
        'options' => [],
        'colonnes' => 3,
      ],
    ],
  ];
}
```

```twig
{# templates/views-view-carte.html.twig #}
<div class="carte-grille carte-grille--{{ colonnes }}-col">
  {% for row in rows %}
    <div class="carte-grille__item">
      {{ row.content }}
    </div>
  {% endfor %}
</div>
```

---

## Row Plugin Custom

```php
<?php
// src/Plugin/views/row/CarteRow.php
namespace Drupal\mon_module\Plugin\views\row;

use Drupal\views\Plugin\views\row\RowPluginBase;
use Drupal\views\ResultRow;

/**
 * Rendu d'une ligne comme carte.
 *
 * @ViewsRow(
 *   id = "mon_module_carte_row",
 *   title = @Translation("Carte"),
 *   help = @Translation("Affiche chaque résultat comme une carte."),
 *   theme = "views_view_row_carte",
 *   display_types = {"normal"}
 * )
 */
class CarteRow extends RowPluginBase {

  /**
   * Rendu d'une ligne.
   */
  public function render(ResultRow $row): array {
    $entity = $row->_entity;

    return [
      '#theme' => 'views_view_row_carte',
      '#entity' => $entity,
      '#view' => $this->view,
      '#fields' => $this->view->field,
      '#row' => $row,
    ];
  }
}
```

---

## Display Plugin Custom

```php
<?php
// src/Plugin/views/display/ICalDisplay.php
namespace Drupal\mon_module\Plugin\views\display;

use Drupal\views\Plugin\views\display\DisplayPluginBase;
use Symfony\Component\HttpFoundation\Response;

/**
 * Display Plugin iCalendar — exporte les résultats en format .ics.
 *
 * @ViewsDisplay(
 *   id = "mon_module_ical",
 *   title = @Translation("iCalendar export"),
 *   help = @Translation("Exporte les résultats en format iCal (.ics)."),
 *   uses_menu_links = FALSE,
 *   uses_route = TRUE,
 *   admin = @Translation("iCal export")
 * )
 */
class ICalDisplay extends DisplayPluginBase {

  /**
   * Définir les options spécifiques à ce display.
   */
  protected function defineOptions(): array {
    $options = parent::defineOptions();
    $options['path'] = ['default' => 'calendrier.ics'];
    return $options;
  }

  /**
   * Le display peut-il afficher des champs ?
   */
  public function usesFields(): bool {
    return TRUE;
  }

  /**
   * Rendu final — retourner une réponse HTTP iCal.
   */
  public function render(): Response {
    $this->view->build();
    $this->view->execute();

    $ical_content = "BEGIN:VCALENDAR\r\nVERSION:2.0\r\n";

    foreach ($this->view->result as $row) {
      $entity = $row->_entity;
      if (!$entity) { continue; }

      $start = $entity->get('field_date')->value;
      $title = $entity->getTitle();

      $ical_content .= "BEGIN:VEVENT\r\n";
      $ical_content .= "DTSTART:" . str_replace(['-', ':'], '', $start) . "\r\n";
      $ical_content .= "SUMMARY:" . $title . "\r\n";
      $ical_content .= "UID:" . $entity->uuid() . "\r\n";
      $ical_content .= "END:VEVENT\r\n";
    }

    $ical_content .= "END:VCALENDAR\r\n";

    return new Response(
      $ical_content,
      200,
      [
        'Content-Type' => 'text/calendar; charset=utf-8',
        'Content-Disposition' => 'attachment; filename="calendrier.ics"',
      ]
    );
  }

  /**
   * Déclarer la route pour ce display.
   */
  public function getRoutedDisplay(): bool {
    return TRUE;
  }
}
```

---

## Déclarer les Plugins dans le Module

```yaml
# mon_module.services.yml — pas nécessaire pour les views plugins
# Les Views plugins sont découverts automatiquement via les annotations/attributes
# dans src/Plugin/views/{type}/

# Pour vérifier la découverte :
# drush php:eval "var_dump(\Drupal::service('views.views_data')->getAll());" | grep mon_module
```

```php
// Vérifier que le plugin est découvert
// drush php:eval "
// $manager = \Drupal::service('plugin.manager.views.style');
// var_dump(array_keys($manager->getDefinitions()));
// "
```
