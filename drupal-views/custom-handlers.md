---
name: drupal-views — custom handlers
description: Créer des handlers Views custom (FieldHandler, FilterHandler, SortHandler, RelationshipHandler, AreaHandler) avec les attributs PHP D11 et les patterns corrects de batch loading, cache, et query alteration.
---

# Handlers Views Custom — Référence Complète

## Architecture des Handlers

```
Views Plugin System
├── FieldHandler       → Affiche une valeur dans une colonne Views
├── FilterHandler      → Filtre les résultats (WHERE clause)
├── SortHandler        → Tri des résultats (ORDER BY clause)
├── RelationshipHandler → Jointure entre tables (JOIN clause)
├── AreaHandler        → Zone header/footer/empty de la View
└── ArgumentHandler    → Contextual filter (argument URL)
```

Tous héritent de classes base dans `Drupal\views\Plugin\views\` et utilisent le système de plugins Drupal.

---

## FieldHandler Custom

```php
<?php
// src/Plugin/views/field/CommandeMontant.php
namespace Drupal\mon_module\Plugin\views\field;

use Drupal\views\Plugin\views\field\FieldPluginBase;
use Drupal\views\ResultRow;
use Drupal\Core\Form\FormStateInterface;

/**
 * Affiche le montant d'une commande formaté en euros.
 *
 * @ViewsField("mon_module_commande_montant")
 */
// D11 : remplacer l'annotation par un attribute PHP :
// #[ViewsField("mon_module_commande_montant")]
class CommandeMontant extends FieldPluginBase {

  /**
   * Options du handler — configuration de l'affichage.
   */
  protected function defineOptions(): array {
    $options = parent::defineOptions();
    $options['devise'] = ['default' => 'EUR'];
    $options['afficher_symbole'] = ['default' => TRUE];
    return $options;
  }

  /**
   * Formulaire de configuration dans l'UI Views.
   */
  public function buildOptionsForm(&$form, FormStateInterface $form_state): void {
    parent::buildOptionsForm($form, $form_state);

    $form['devise'] = [
      '#type' => 'select',
      '#title' => $this->t('Devise'),
      '#options' => ['EUR' => '€ (EUR)', 'USD' => '$ (USD)'],
      '#default_value' => $this->options['devise'],
    ];

    $form['afficher_symbole'] = [
      '#type' => 'checkbox',
      '#title' => $this->t('Afficher le symbole de devise'),
      '#default_value' => $this->options['afficher_symbole'],
    ];
  }

  /**
   * Rendu de la valeur — appelé pour chaque ligne de résultats.
   */
  public function render(ResultRow $values): array|\Drupal\Component\Render\MarkupInterface|string {
    $montant_centimes = $this->getValue($values);

    if ($montant_centimes === NULL) {
      return '';
    }

    $montant = $montant_centimes / 100;
    $devise = $this->options['devise'];
    $symbole = $this->options['afficher_symbole'];

    $formatted = number_format($montant, 2, ',', ' ');

    if ($symbole) {
      $symbols = ['EUR' => '€', 'USD' => '$'];
      $formatted = $formatted . ' ' . ($symbols[$devise] ?? $devise);
    }

    return $formatted;
  }

  /**
   * Ajouter les colonnes nécessaires à la query.
   * Appelé avant l'exécution pour déclarer les champs SELECT.
   */
  public function query(): void {
    // Appeler parent pour ajouter le champ de base (montant)
    $this->ensureMyTable();
    $this->addAdditionalFields();

    // Si vous avez besoin de colonnes supplémentaires de la même table :
    // $this->field_alias = $this->query->addField($this->tableAlias, 'montant');
  }

  /**
   * Batch loading — pour les entités référencées.
   * Charger toutes les entités d'un coup au lieu de N requêtes.
   */
  public function preRender(array $values): void {
    // Collecter tous les IDs
    $ids = [];
    foreach ($values as $row) {
      $id = $this->getValue($row, 'commande_id');
      if ($id) {
        $ids[] = $id;
      }
    }

    if ($ids) {
      // Charger toutes les entités en une seule requête
      $this->commandes = \Drupal::entityTypeManager()
        ->getStorage('mon_module_commande')
        ->loadMultiple($ids);
    }
  }
}
```

---

## FilterHandler Custom

```php
<?php
// src/Plugin/views/filter/CommandeStatut.php
namespace Drupal\mon_module\Plugin\views\filter;

use Drupal\Core\Form\FormStateInterface;
use Drupal\views\Plugin\views\filter\InOperator;

/**
 * Filtre Views sur le statut des commandes.
 *
 * @ViewsFilter("mon_module_commande_statut")
 */
class CommandeStatut extends InOperator {

  /**
   * Valeurs disponibles pour le filtre.
   */
  public function getValueOptions(): void {
    if (!isset($this->valueOptions)) {
      $this->valueOptions = [
        'pending'   => $this->t('En attente'),
        'confirmed' => $this->t('Confirmée'),
        'shipped'   => $this->t('Expédiée'),
        'cancelled' => $this->t('Annulée'),
      ];
    }
  }
}
```

**FilterHandler qui modifie la WHERE clause :**

```php
<?php
// src/Plugin/views/filter/CommandeMontantMin.php
namespace Drupal\mon_module\Plugin\views\filter;

use Drupal\Core\Form\FormStateInterface;
use Drupal\views\Plugin\views\filter\NumericFilter;

/**
 * @ViewsFilter("mon_module_montant_min")
 */
class CommandeMontantMin extends NumericFilter {

  /**
   * Modifier la WHERE clause de la query Views.
   */
  public function query(): void {
    $this->ensureMyTable();

    // Convertir la valeur du filtre (euros) en centimes pour la DB
    $valeur_centimes = (int) ($this->value['value'] * 100);

    $this->query->addWhere(
      $this->options['group'],
      "$this->tableAlias.montant",
      $valeur_centimes,
      '>='
    );
  }

  /**
   * Validation du formulaire exposé.
   */
  public function validateExposed(&$form, FormStateInterface $form_state): void {
    $value = $form_state->getValue($this->options['expose']['identifier']);
    if ($value !== '' && !is_numeric($value)) {
      $form_state->setError(
        $form[$this->options['expose']['identifier']],
        $this->t('Le montant minimum doit être un nombre.')
      );
    }
  }
}
```

---

## SortHandler Custom

```php
<?php
// src/Plugin/views/sort/CommandeScore.php
namespace Drupal\mon_module\Plugin\views\sort;

use Drupal\views\Plugin\views\sort\SortPluginBase;

/**
 * Tri des commandes par score de priorité calculé.
 *
 * @ViewsSort("mon_module_commande_score")
 */
class CommandeScore extends SortPluginBase {

  /**
   * Modifier la ORDER BY clause.
   */
  public function query(): void {
    $this->ensureMyTable();

    // Tri par expression SQL calculée
    $formula = "({$this->tableAlias}.montant * 0.4 + {$this->tableAlias}.priorite * 0.6)";

    $this->query->addOrderBy(
      NULL,                              // NULL = utiliser la formula
      NULL,
      $this->options['order'],           // ASC ou DESC
      'commande_score',                  // Alias de la colonne ORDER BY
      ['function' => $formula]
    );
  }
}
```

---

## RelationshipHandler Custom

```php
<?php
// src/Plugin/views/relationship/CommandeToClient.php
namespace Drupal\mon_module\Plugin\views\relationship;

use Drupal\views\Plugin\views\relationship\RelationshipPluginBase;

/**
 * Relation Views de Commande vers Client (table personnalisée).
 *
 * @ViewsRelationship("mon_module_commande_to_client")
 */
class CommandeToClient extends RelationshipPluginBase {

  /**
   * La table jointe et le champ de jointure sont définis dans
   * hook_views_data() via la clé 'relationship'.
   * Ce handler peut surcharger buildOptionsForm() pour des options custom.
   */
  public function query(): void {
    // Laisser le parent gérer le JOIN standard
    parent::query();

    // Optionnel : ajouter une condition sur la jointure
    // $this->query->addJoin('LEFT', 'mon_module_clients', 'mon_module_clients', ...);
  }
}
```

---

## AreaHandler Custom (Header / Footer / Empty)

```php
<?php
// src/Plugin/views/area/CommandeResume.php
namespace Drupal\mon_module\Plugin\views\area;

use Drupal\views\Plugin\views\area\AreaPluginBase;
use Drupal\views\ResultRow;

/**
 * Affiche un résumé des commandes dans le header Views.
 *
 * @ViewsArea("mon_module_commande_resume")
 */
class CommandeResume extends AreaPluginBase {

  /**
   * Rendu de la zone header/footer.
   * $empty = TRUE si la View n'a aucun résultat (zone "no results").
   */
  public function render($empty = FALSE): array {
    if ($empty) {
      return [
        '#markup' => $this->t('Aucune commande trouvée.'),
      ];
    }

    // Accéder aux résultats de la View
    $view = $this->view;
    $result = $view->result;

    $total_montant = array_sum(array_map(
      fn(ResultRow $row) => $row->mon_module_commandes_montant ?? 0,
      $result
    ));

    return [
      '#theme' => 'mon_module_commande_resume',
      '#total' => count($result),
      '#montant' => $total_montant / 100,
      '#cache' => [
        'tags' => ['mon_module_commande_list'],
        'contexts' => ['user.roles'],
      ],
    ];
  }
}
```

---

## Déclarer les Handlers dans hook_views_data

```php
// Dans hook_views_data() — référencer les handlers custom par leur @ViewsField ID
$data['mon_module_commandes']['montant'] = [
  'title' => t('Montant formaté'),
  'field' => [
    'id' => 'mon_module_commande_montant',   // ID du #[ViewsField] / @ViewsField
  ],
  'filter' => [
    'id' => 'mon_module_montant_min',
    'title' => t('Montant minimum'),
  ],
  'sort' => [
    'id' => 'mon_module_commande_score',
  ],
];

// Area handler — déclaré séparément (pas lié à une colonne)
$data['mon_module_commandes']['resume_area'] = [
  'title' => t('Résumé des commandes'),
  'help' => t('Affiche le total des commandes dans le header.'),
  'area' => [
    'id' => 'mon_module_commande_resume',
  ],
];
```

---

## Cache dans les Handlers

```php
// Dans un FieldHandler — déclarer les cache tags pour la valeur rendue
public function getCacheTags(): array {
  return Cache::mergeTags(
    parent::getCacheTags(),
    ['mon_module_commande_list']
  );
}

public function getCacheContexts(): array {
  return Cache::mergeContexts(
    parent::getCacheContexts(),
    ['user.roles']
  );
}
```
