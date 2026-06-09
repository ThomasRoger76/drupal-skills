---
name: drupal-cron-queue — queue workers
description: Créer des QueueWorker plugins Drupal pour le traitement asynchrone - plugin QueueWorker, processItem(), retry, SuspendQueueException, et commandes Drush.
---

# Queue Workers — Référence Complète

## Créer un QueueWorker Plugin

```php
<?php
// src/Plugin/QueueWorker/RssImportWorker.php
namespace Drupal\mon_module\Plugin\QueueWorker;

use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\Core\Queue\Attribute\QueueWorker;
use Drupal\Core\Queue\QueueWorkerBase;
use Drupal\Core\Queue\SuspendQueueException;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use GuzzleHttp\ClientInterface;
use Psr\Log\LoggerInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Traite les imports RSS en arrière-plan.
 *
 * D10.2+ / D11 : attribut PHP standard (l'annotation @QueueWorker est dépréciée).
 * cron.time = budget max en secondes consacré à cette queue par run de cron.
 */
#[QueueWorker(
  id: 'mon_module_rss_import',
  title: new TranslatableMarkup('Import RSS'),
  cron: ['time' => 60],
)]
class RssImportWorker extends QueueWorkerBase implements ContainerFactoryPluginInterface {

  public function __construct(
    array $configuration,
    string $plugin_id,
    mixed $plugin_definition,
    private readonly ClientInterface $httpClient,
    private readonly LoggerInterface $logger,
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);
  }

  public static function create(ContainerInterface $container, array $configuration, $plugin_id, $plugin_definition): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('http_client'),
      $container->get('logger.channel.mon_module'),
    );
  }

  /**
   * Traiter UN item de la queue.
   *
   * Si cette méthode :
   *   - se termine normalement → item supprimé de la queue (succès)
   *   - lance une Exception → item remis en queue (retry)
   *   - lance SuspendQueueException → toute la queue est suspendue
   */
  public function processItem($data): void {
    $url = $data['url'] ?? NULL;

    if (!$url) {
      // Données invalides → ne pas retenter
      $this->logger->warning('Item RSS sans URL ignoré.');
      return;  // Pas d'exception = item supprimé définitivement
    }

    try {
      $response = $this->httpClient->get($url, ['timeout' => 10]);

      if ($response->getStatusCode() !== 200) {
        throw new \RuntimeException('HTTP ' . $response->getStatusCode());
      }

      $xml = simplexml_load_string($response->getBody()->getContents());
      // ... traitement du XML ...

      $this->logger->info('RSS importé depuis @url', ['@url' => $url]);
    }
    catch (\GuzzleHttp\Exception\ConnectException $e) {
      // Erreur réseau temporaire → suspendre TOUTE la queue (API down)
      throw new SuspendQueueException(
        'API RSS inaccessible : ' . $e->getMessage()
      );
    }
    catch (\Exception $e) {
      // Autre erreur → item remis en queue (sera retenté)
      $this->logger->error('Erreur import @url : @error', [
        '@url' => $url,
        '@error' => $e->getMessage(),
      ]);
      throw $e;  // Re-lancer = retry automatique
    }
  }
}
```

---

## Ajouter des Items à une Queue

```php
// Ajouter un item à la queue depuis n'importe quel code Drupal
$queue = \Drupal::queue('mon_module_rss_import');

// Item simple
$queue->createItem(['url' => 'https://example.com/feed.rss']);

// Item structuré
$queue->createItem([
  'url' => 'https://example.com/feed.rss',
  'entity_id' => 42,
  'operation' => 'update',
  'priority' => 'high',
]);

// Nombre d'items en attente
$count = $queue->numberOfItems();
echo "$count items en queue";

// Ajouter depuis hook_cron (pattern recommandé)
function mon_module_cron(): void {
  $items_to_process = fetch_items_needing_processing();

  $queue = \Drupal::queue('mon_module_rss_import');
  foreach ($items_to_process as $item) {
    $queue->createItem(['id' => $item->id, 'url' => $item->url]);
  }
}
```

---

## Commandes Drush Queue

```bash
# Lister toutes les queues et leur taille
drush queue:list

# Traiter une queue complète (jusqu'à vider ou timeout)
drush queue:run mon_module_rss_import

# Traiter avec time limit (secondes)
drush queue:run mon_module_rss_import --time-limit=300

# Traiter N items maximum
drush queue:run mon_module_rss_import --items-limit=50

# Vider une queue (sans traiter les items)
drush queue:delete mon_module_rss_import

# Voir le nombre d'items dans une queue
drush php:eval "
echo \Drupal::queue('mon_module_rss_import')->numberOfItems() . ' items en attente';
"

# En production — lancer en background
nohup drush queue:run mon_module_rss_import --time-limit=3600 > /tmp/queue.log 2>&1 &
```

---

## Retry et Gestion des Erreurs

```php
// Comportements possibles dans processItem() :
// 1. return → item supprimé (succès ou abandon volontaire)
// 2. throw \Exception → item remis en queue après expiration du lease (retry au prochain run)
// 3. throw RequeueException → item remis en queue IMMÉDIATEMENT (rejouable dès ce run)
// 4. throw DelayedRequeueException → item verrouillé N secondes avant retry (backoff)
//    Nécessite une queue implémentant DelayableQueueInterface (DatabaseQueue le fait).
// 5. throw SuspendQueueException → queue entière suspendue pour ce run (service externe down)

// Pattern de retry avec compteur
public function processItem($data): void {
  $attempts = $data['attempts'] ?? 0;

  if ($attempts >= 3) {
    // Trop de tentatives → abandon + log
    $this->logger->error('Item abandonné après 3 tentatives : @id', ['@id' => $data['id']]);
    return;  // Pas d'exception = supprimé définitivement
  }

  try {
    // ... traitement ...
  }
  catch (\Exception $e) {
    // Incrémenter le compteur et remettre en queue
    $data['attempts'] = $attempts + 1;
    \Drupal::queue('mon_module_rss_import')->createItem($data);
    // Ne pas relancer l'exception (item original sera supprimé après return)
    $this->logger->warning('Retry @n pour @id', ['@n' => $data['attempts'], '@id' => $data['id'] ?? '?']);
  }
}
```

```php
// Backoff natif sans recréer l'item : DelayedRequeueException (D9.3+).
// L'item reste en queue, verrouillé pendant le délai → pas de retry en boucle serrée.
use Drupal\Core\Queue\DelayedRequeueException;

public function processItem($data): void {
  try {
    $this->callExternalApi($data);
  }
  catch (\GuzzleHttp\Exception\ServerException $e) {
    // 5xx temporaire : ne pas marmarteler l'API, attendre 5 min avant retry.
    throw new DelayedRequeueException(300, 'API 5xx, retry différé : ' . $e->getMessage());
  }
}
```
