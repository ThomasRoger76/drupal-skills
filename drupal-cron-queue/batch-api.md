---
name: drupal-cron-queue — batch api
description: Traiter des milliers d'items Drupal sans timeout avec la Batch API - batch_set(), opérations, finished callback, barre de progression, et Batch depuis Drush.
---

# Batch API — Référence Complète

## Quand Utiliser la Batch API

```
Queue API :
  → Items traités en arrière-plan (asynchrone)
  → Pas d'interface utilisateur visible
  → Items persist entre les runs de cron

Batch API :
  → Traitement en foreground avec barre de progression
  → Parfait pour les opérations déclenchées manuellement
  → Contexte persistant entre les chunks ($context['sandbox'])
  → Retour vers l'utilisateur à la fin (finished callback)
  → Depuis Drush : `drush_backend_batch_process()`
```

---

## batch_set() — Démarrer un Batch

```php
<?php

use Drupal\Core\Url;

/**
 * Déclencher un batch depuis un formulaire ou Controller.
 */
class ImportController extends ControllerBase {

  public function lancerImport(): array|Response {
    // Récupérer les IDs à traiter
    $nids = \Drupal::entityQuery('node')
      ->condition('type', 'article')
      ->condition('status', 1)
      ->accessCheck(FALSE)
      ->execute();

    // Diviser en chunks de 50
    $chunks = array_chunk(array_values($nids), 50);

    // Créer les opérations batch
    $operations = [];
    foreach ($chunks as $chunk) {
      $operations[] = [
        'Drupal\mon_module\Batch\ArticleBatch::processChunk',
        [$chunk, count($nids)],  // Arguments passés à processChunk
      ];
    }

    // Configurer le batch
    $batch = [
      'title' => $this->t('Import des articles en cours...'),
      'operations' => $operations,
      'finished' => 'Drupal\mon_module\Batch\ArticleBatch::finished',
      'init_message' => $this->t('Initialisation...'),
      'progress_message' => $this->t('Traitement des articles : @current sur @total.'),
      'error_message' => $this->t('Une erreur est survenue.'),
    ];

    batch_set($batch);

    // Retourner au formulaire ou à une page cible après le batch
    return batch_process(Url::fromRoute('mon_module.import_results'));
  }
}
```

---

## Classe de Traitement Batch

```php
<?php
// src/Batch/ArticleBatch.php
namespace Drupal\mon_module\Batch;

use Drupal\node\Entity\Node;

class ArticleBatch {

  /**
   * Opération de traitement d'un chunk d'articles.
   *
   * @param array $nids      IDs des nœuds dans ce chunk
   * @param int   $total     Nombre total d'articles à traiter
   * @param array $context   Contexte Batch (sandbox, results, message)
   */
  public static function processChunk(array $nids, int $total, array &$context): void {
    // Initialiser le sandbox (persistant entre les opérations)
    if (empty($context['sandbox'])) {
      $context['sandbox']['progress'] = 0;
      $context['sandbox']['total'] = $total;
    }

    // Traiter ce chunk
    $nodes = Node::loadMultiple($nids);
    foreach ($nodes as $node) {
      // Logique de traitement
      $node->set('field_traite', TRUE);
      $node->save();

      $context['sandbox']['progress']++;
      $context['results']['processed'][] = $node->id();
    }

    // Calculer la progression (0 à 1)
    $context['finished'] = $context['sandbox']['progress'] / $context['sandbox']['total'];

    // Message affiché dans la barre de progression
    $context['message'] = t(
      'Traitement : @current / @total articles',
      [
        '@current' => $context['sandbox']['progress'],
        '@total' => $context['sandbox']['total'],
      ]
    );
  }

  /**
   * Callback appelé à la fin du batch (succès ou erreur).
   *
   * @param bool   $success    TRUE si terminé sans erreur
   * @param array  $results    Résultats accumulés dans $context['results']
   * @param array  $operations Opérations non exécutées (si erreur)
   */
  public static function finished(bool $success, array $results, array $operations): void {
    if ($success) {
      $count = count($results['processed'] ?? []);
      \Drupal::messenger()->addStatus(
        t('@count articles traités avec succès.', ['@count' => $count])
      );
    }
    else {
      \Drupal::messenger()->addError(
        t('Le traitement a été interrompu. @count articles traités avant l\'erreur.', [
          '@count' => count($results['processed'] ?? []),
        ])
      );
    }
  }
}
```

---

## Batch depuis Drush (CLI)

```php
// Dans une commande Drush custom
use Drush\Commands\DrushCommands;

class ImportCommands extends DrushCommands {

  /**
   * @command mon_module:import
   * @aliases mm-import
   */
  public function import(): void {
    $nids = \Drupal::entityQuery('node')
      ->condition('type', 'article')
      ->accessCheck(FALSE)
      ->execute();

    $chunks = array_chunk(array_values($nids), 50);
    $operations = [];
    foreach ($chunks as $chunk) {
      $operations[] = ['Drupal\mon_module\Batch\ArticleBatch::processChunk', [$chunk, count($nids)]];
    }

    $batch = [
      'operations' => $operations,
      'finished' => 'Drupal\mon_module\Batch\ArticleBatch::finished',
    ];

    batch_set($batch);

    // Indispensable pour exécuter le batch depuis Drush
    drush_backend_batch_process();
  }
}
```

```bash
# Exécuter le batch depuis Drush
drush mon_module:import
# → Affiche la progression : [=====>   ] 50% (250 / 500)
```

---

## Batch avec Données Persistantes entre Chunks

```php
// Pattern pour traiter une source de données externe
public static function processChunk(int $offset, int $batch_size, array &$context): void {
  // Initialiser le sandbox
  if (!isset($context['sandbox']['current_offset'])) {
    $context['sandbox']['current_offset'] = 0;
    $context['sandbox']['total'] = self::countTotalItems();
  }

  // Charger le chunk depuis la source
  $items = self::fetchItems($context['sandbox']['current_offset'], $batch_size);

  foreach ($items as $item) {
    self::processItem($item);
    $context['results']['count'] = ($context['results']['count'] ?? 0) + 1;
  }

  // Avancer l'offset
  $context['sandbox']['current_offset'] += count($items);

  // Progression
  $total = $context['sandbox']['total'];
  $done = $context['sandbox']['current_offset'];
  $context['finished'] = $total > 0 ? min(1.0, $done / $total) : 1.0;
  $context['message'] = t('@done / @total items traités', ['@done' => $done, '@total' => $total]);

  // Cas où il n'y a plus rien à faire
  if (empty($items)) {
    $context['finished'] = 1.0;
  }
}
```
