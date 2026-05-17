---
name: drupal-views — programmatic
description: Charger et exécuter des Views depuis PHP, modifier les queries via hooks, altérer les résultats, debugger les queries SQL, et construire des Views programmatiquement via l'API.
---

# Views Programmatique — API PHP & Hooks

## Charger et Exécuter une View

```php
<?php

use Drupal\views\Views;

// ── Méthode 1 : obtenir le résultat (tableau de ResultRow) ─────────────────
$view = Views::getView('ma_view');

// Toujours vérifier que la View existe et est active
if (!$view || !$view->access('page_1')) {
  // La View n'existe pas ou l'utilisateur n'y a pas accès
  return [];
}

// Définir le display à utiliser
$view->setDisplay('page_1');   // 'default', 'page_1', 'block_1', etc.

// Passer des arguments (contextual filters) si nécessaire
$view->setArguments([42]);      // Premier contextual filter = 42

// Définir des filtres exposés programmatiquement
$view->setExposedInput([
  'statut' => 'confirmed',
  'type' => 'article',
]);

// Exécuter la query
$view->execute();

// Accéder aux résultats
$results = $view->result;       // Array de ResultRow objects

foreach ($results as $row) {
  // Accéder aux valeurs brutes
  $nid = $row->nid;
  $title = $row->node_field_data_title;   // alias Views

  // Accéder à l'entité chargée (si la View charge des entités)
  $entity = $row->_entity;

  // Rendre un champ
  $field_output = $view->field['title']->advancedRender($row);
}

// ── Méthode 2 : obtenir le render array complet ───────────────────────────
$view2 = Views::getView('ma_view');
if ($view2 && $view2->access('block_1')) {
  $view2->setDisplay('block_1');
  $render_array = $view2->render();
  // $render_array est prêt pour un render() ou return dans un build()
}

// ── Méthode 3 : embed dans un render array (recommandé pour les blocs) ─────
$render = [
  '#type' => 'view',
  '#name' => 'ma_view',
  '#display_id' => 'block_1',
  '#arguments' => [42],
  '#embed' => TRUE,          // TRUE = pas de page entière, juste le contenu
];
```

---

## hook_views_query_alter — Modifier la Query SQL

```php
<?php

/**
 * Implements hook_views_query_alter().
 *
 * Appelé AVANT l'exécution — modifier la query SQL de toutes les Views.
 */
function mon_module_views_query_alter(\Drupal\views\ViewExecutable $view, \Drupal\views\Plugin\views\query\QueryPluginBase $query): void {
  // Cibler une View spécifique
  if ($view->id() !== 'commandes_liste') {
    return;
  }

  // $query est un objet Sql — accéder à la SelectQuery sous-jacente
  if ($query instanceof \Drupal\views\Plugin\views\query\Sql) {
    // Ajouter une condition WHERE custom
    $query->addWhere(
      0,                                         // Groupe de conditions (0 = défaut)
      'mon_module_commandes.montant',
      500,                                       // Valeur
      '>='                                       // Opérateur
    );

    // Ajouter un ORDER BY custom
    $query->addOrderBy(NULL, NULL, 'ASC', 'commandes_liste_montant');

    // Accéder à la SelectQuery sous-jacente pour des modifications avancées
    // Disponible uniquement APRÈS $view->build() — pas dans query_alter
  }
}
```

---

## hook_views_pre_build / hook_views_post_build

```php
/**
 * Implements hook_views_pre_build().
 *
 * Avant la construction de la query — modifier les arguments ou les filtres.
 */
function mon_module_views_pre_build(\Drupal\views\ViewExecutable $view): void {
  if ($view->id() === 'commandes_liste') {
    // Forcer un filtre exposé selon le rôle de l'utilisateur
    $current_user = \Drupal::currentUser();
    if (!$current_user->hasPermission('voir toutes les commandes')) {
      $view->setExposedInput(
        array_merge(
          $view->getExposedInput(),
          ['uid' => $current_user->id()]
        )
      );
    }
  }
}
```

---

## hook_views_post_execute — Modifier les Résultats

```php
/**
 * Implements hook_views_post_execute().
 *
 * Après l'exécution, avant le rendu — modifier $view->result.
 */
function mon_module_views_post_execute(\Drupal\views\ViewExecutable $view): void {
  if ($view->id() !== 'commandes_liste') {
    return;
  }

  // Enrichir les résultats avec des données calculées
  foreach ($view->result as $row) {
    // Ajouter une propriété custom à la ResultRow
    $row->tva = ($row->mon_module_commandes_montant ?? 0) * 0.20 / 100;
    $row->montant_ttc = ($row->mon_module_commandes_montant ?? 0) * 1.20 / 100;
  }

  // Filtrer les résultats (utiliser avec précaution — ça casse la pagination)
  // Mieux : utiliser hook_views_query_alter pour filtrer en SQL
  // $view->result = array_filter($view->result, fn($row) => $row->montant > 100);
}
```

---

## hook_views_pre_render — Avant le Rendu

```php
/**
 * Implements hook_views_pre_render().
 *
 * Modifier l'objet View avant le rendu HTML — pas les données.
 * Idéal pour ajouter des cache tags, modifier les attachments JS/CSS.
 */
function mon_module_views_pre_render(\Drupal\views\ViewExecutable $view): void {
  if ($view->id() !== 'commandes_liste') {
    return;
  }

  // Ajouter des cache tags custom
  $view->element['#cache']['tags'][] = 'mon_module_commande_list';
  $view->element['#cache']['tags'][] = 'user:' . \Drupal::currentUser()->id();

  // Attacher une library JS/CSS
  $view->element['#attached']['library'][] = 'mon_module/commandes-view';

  // Passer des données à drupalSettings (accessible en JS)
  $view->element['#attached']['drupalSettings']['monModule']['commandesCount']
    = count($view->result);
}
```

---

## Déboguer une Query Views

```php
/**
 * Implements hook_views_query_alter().
 * Utiliser UNIQUEMENT en développement — jamais en production.
 */
function mon_module_views_query_alter(\Drupal\views\ViewExecutable $view, $query): void {
  if ($view->id() !== 'commandes_liste') {
    return;
  }

  // Afficher la query SQL dans le HTML (avec Devel module)
  if (\Drupal::moduleHandler()->moduleExists('devel') && function_exists('dpq')) {
    dpq($view->query);    // Dump de la query objet Views
  }

  // Alternative : logger la query SQL
  // Disponible dans hook_views_post_execute :
  // \Drupal::logger('mon_module')->debug($view->query->query());
}

/**
 * Dans hook_views_post_execute — afficher la query SQL réelle.
 */
function mon_module_views_post_execute(\Drupal\views\ViewExecutable $view): void {
  if ($view->id() !== 'commandes_liste') {
    return;
  }

  // Récupérer la query SQL exécutée (avec placeholders)
  if (isset($view->query->query)) {
    \Drupal::logger('mon_module')->debug(
      'Views query: @query',
      ['@query' => (string) $view->query->query]
    );
  }
}
```

---

## Compter les Résultats sans Rendu

```php
// Compter rapidement sans charger les entités
function mon_module_count_commandes_actives(): int {
  $view = Views::getView('commandes_liste');
  if (!$view) {
    return 0;
  }

  $view->setDisplay('default');
  $view->setArguments([]);

  // Utiliser get_total_rows pour la pagination
  $view->get_total_rows = TRUE;
  $view->execute();

  return $view->total_rows ?? count($view->result);
}
```

---

## Construire une View via l'API (avancé)

```php
<?php
// Créer une View config entity programmatiquement
use Drupal\views\Entity\View;

$view = View::create([
  'id' => 'mes_commandes_api',
  'label' => 'Mes commandes (API)',
  'base_table' => 'mon_module_commandes',
  'description' => 'View créée programmatiquement.',
]);

// Configurer le display par défaut
$display = &$view->getDisplay('default');
$display['display_options']['fields']['reference'] = [
  'id' => 'reference',
  'table' => 'mon_module_commandes',
  'field' => 'reference',
  'plugin_id' => 'standard',
];
$display['display_options']['filters']['statut'] = [
  'id' => 'statut',
  'table' => 'mon_module_commandes',
  'field' => 'statut',
  'value' => ['confirmed' => 'confirmed'],
  'plugin_id' => 'in_operator',
];

$view->save();
```
