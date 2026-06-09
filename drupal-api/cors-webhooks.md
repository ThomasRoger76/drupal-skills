---
name: drupal-api — CORS et webhooks
description: Configuration CORS pour les frontends découplés, implémentation de webhooks sortants depuis Drupal, et création de REST Resource plugins custom.
---

# CORS & Webhooks — Référence Complète

## Configuration CORS (services.yml)

```yaml
# web/sites/default/services.yml

parameters:
  cors.config:
    enabled: true

    # Headers autorisés dans les requêtes CORS
    allowedHeaders:
      - 'x-csrf-token'      # Pour les mutations JSON:API
      - 'authorization'     # Bearer token OAuth
      - 'content-type'      # application/vnd.api+json
      - 'accept'
      - 'origin'
      - 'x-requested-with'
      - 'cache-control'

    # Méthodes HTTP autorisées
    allowedMethods:
      - 'GET'
      - 'POST'
      - 'PATCH'
      - 'DELETE'
      - 'PUT'
      - 'OPTIONS'

    # Origines autorisées — JAMAIS '*' en production
    allowedOrigins:
      - 'https://mon-frontend.com'
      - 'https://staging.mon-frontend.com'
      - 'http://localhost:3000'      # Next.js dev
      - 'http://localhost:3001'      # Alternative dev port

    # Origines avec regex (plus flexible)
    allowedOriginsPatterns:
      - '/^https?:\/\/(localhost|127\.0\.0\.1)(:\d+)?$/'
      - '/^https:\/\/.*\.mon-domaine\.com$/'   # Tous les sous-domaines

    # Headers exposés au client (pour debugging)
    exposedHeaders:
      - 'X-Drupal-Cache-Tags'
      - 'X-Drupal-Cache-Contexts'

    maxAge: 3600           # Cache de la preflight (OPTIONS) en secondes
    supportsCredentials: true   # Requis pour les cookies de session

# Après modification → drush cr obligatoire (docker compose exec php drush cr)
```

> **CORS par environnement.** Ne jamais committer `localhost`/`*` pour la prod. Garder une whitelist
> stricte par environnement : surcharger `$config['cors.config']['allowedOrigins']` dans `settings.php`
> via `getenv('CORS_ALLOWED_ORIGINS')`, ou ajouter les origins de dev uniquement dans
> `settings.local.php` (non commité). Voir [authentication.md](authentication.md#configuration-cors-pour-le-frontend).

**Vérifier CORS :**
```bash
curl -X OPTIONS https://mon-drupal.com/jsonapi/node/article \
  -H "Origin: https://mon-frontend.com" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: Authorization, Content-Type" \
  -v 2>&1 | grep -i "access-control"
```

---

## Webhooks Sortants — Notifier le Frontend

### Sur sauvegarde d'entité

```php
// src/EventSubscriber/ContenuWebhookSubscriber.php
namespace Drupal\mon_module\EventSubscriber;

use Drupal\Core\Entity\EntityInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use GuzzleHttp\ClientInterface;
use Psr\Log\LoggerInterface;

// ⚠️ DEUX APPROCHES — choisir selon le projet :
//
// APPROCHE A (recommandée) : hook Drupal dans .module
// → Simple, natif, aucune dépendance
// function mon_module_node_update(NodeInterface $node): void { ... }
//
// APPROCHE B : EventSubscriber via drupal/hook_event_dispatcher (contrib requis)
// → composer require drupal/hook_event_dispatcher
// → Plus testable (mock du dispatcher)
// → Utilise l'EventDispatcher Symfony

// ─── APPROCHE B — EventSubscriber (nécessite drupal/hook_event_dispatcher) ──

use Drupal\core_event_dispatcher\Event\Entity\EntityInsertEvent;
use Drupal\core_event_dispatcher\Event\Entity\EntityUpdateEvent;
use Drupal\core_event_dispatcher\Event\Entity\EntityDeleteEvent;
use Drupal\core_event_dispatcher\EntityHookEvents;

class ContenuWebhookSubscriber implements EventSubscriberInterface {

  public function __construct(
    private readonly ClientInterface $httpClient,
    private readonly LoggerInterface $logger,
  ) {}

  public static function getSubscribedEvents(): array {
    return [
      // Ces events existent SEULEMENT si drupal/hook_event_dispatcher est installé
      EntityHookEvents::ENTITY_INSERT => 'onEntityChange',
      EntityHookEvents::ENTITY_UPDATE => 'onEntityChange',
      EntityHookEvents::ENTITY_DELETE => 'onEntityDelete',
    ];
  }

  public function onEntityChange(EntityInsertEvent|EntityUpdateEvent $event): void {
    $entity = $event->getEntity();
    if (!$entity instanceof \Drupal\node\NodeInterface) {
      return;
    }

    $this->sendWebhook('content.updated', [
      'type' => $entity->bundle(),
      'id' => $entity->id(),
      'uuid' => $entity->uuid(),
      'langcode' => $entity->language()->getId(),
      'status' => $entity->isPublished(),
      'changed' => $entity->getChangedTime(),
    ]);
  }

  private function sendWebhook(string $event, array $data): void {
    $webhook_url = \Drupal::config('mon_module.settings')->get('webhook_url');
    $webhook_secret = \Drupal::config('mon_module.settings')->get('webhook_secret');

    if (!$webhook_url) {
      return;
    }

    $payload = json_encode(['event' => $event, 'data' => $data]);

    // HMAC signature pour sécuriser le webhook
    $signature = 'sha256=' . hash_hmac('sha256', $payload, $webhook_secret);

    try {
      $this->httpClient->post($webhook_url, [
        'json' => ['event' => $event, 'data' => $data],
        'headers' => [
          'X-Webhook-Signature' => $signature,
          'X-Drupal-Event' => $event,
          'Content-Type' => 'application/json',
        ],
        'timeout' => 5,        // Timeout court — ne pas bloquer Drupal
        'connect_timeout' => 2,
      ]);
    }
    catch (\Exception $e) {
      $this->logger->error('Webhook failed: @error', ['@error' => $e->getMessage()]);
      // Ne pas propager l'exception — le webhook ne doit pas bloquer la sauvegarde
    }
  }
}
```

### Webhook avec Queue (recommandé pour la résilience)

```php
// QueueWorker pour les webhooks — retry si le frontend est down
// src/Plugin/QueueWorker/WebhookQueueWorker.php

use Drupal\Core\Annotation\Translation;
use Drupal\Core\Queue\Attribute\QueueWorker;
use Drupal\Core\Queue\QueueWorkerBase;

// D11 : attribut PHP (#[QueueWorker]) — l'annotation @QueueWorker est dépréciée.
#[QueueWorker(
  id: 'mon_module_webhooks',
  title: new Translation('Webhooks sortants'),
  cron: ['time' => 30],
)]
class WebhookQueueWorker extends QueueWorkerBase {
  public function processItem($data): void {
    $this->httpClient->post($data['url'], [
      'json' => $data['payload'],
      'headers' => $data['headers'],
      'timeout' => 10,
    ]);
    // Si exception → Drupal remet en queue automatiquement
  }
}

// Dans l'EventSubscriber — mettre en queue au lieu d'envoyer directement
private function queueWebhook(string $event, array $data): void {
  $queue = \Drupal::queue('mon_module_webhooks');
  $queue->createItem([
    'url' => $this->getWebhookUrl(),
    'payload' => ['event' => $event, 'data' => $data],
    'headers' => ['X-Webhook-Signature' => $this->sign($data)],
  ]);
}
```

---

## REST Resource Custom

```php
<?php
// src/Plugin/rest/resource/StatistiquesResource.php
namespace Drupal\mon_module\Plugin\rest\resource;

use Drupal\Core\Annotation\Translation;
use Drupal\rest\Attribute\RestResource;
use Drupal\rest\Plugin\ResourceBase;
use Drupal\rest\ResourceResponse;
use Drupal\Core\Session\AccountProxyInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Endpoint REST pour les statistiques du site.
 *
 * D11 : attribut PHP (#[RestResource]) — l'annotation @RestResource est dépréciée.
 * Versioning custom : préfixe /api/v1/ (convention API maison).
 */
#[RestResource(
  id: 'mon_module_statistiques',
  label: new Translation('Statistiques'),
  uri_paths: [
    'canonical' => '/api/v1/stats',
    'create' => '/api/v1/stats',
  ],
)]
class StatistiquesResource extends ResourceBase {

  public function __construct(
    array $configuration,
    $plugin_id,
    $plugin_definition,
    array $serializer_formats,
    \Psr\Log\LoggerInterface $logger,
    private readonly AccountProxyInterface $currentUser,
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition, $serializer_formats, $logger);
  }

  public static function create(ContainerInterface $container, array $configuration, $plugin_id, $plugin_definition): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->getParameter('serializer.formats'),
      $container->get('logger.factory')->get('rest'),
      $container->get('current_user'),
    );
  }

  /**
   * GET /api/v1/stats?_format=json
   */
  public function get(): ResourceResponse {
    if (!$this->currentUser->hasPermission('access content')) {
      throw new \Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException();
    }

    $stats = [
      'articles_count' => $this->countPublishedNodes('article'),
      'users_count' => $this->countActiveUsers(),
      // \DateTime::ISO8601 est déprécié (non conforme RFC 3339) → utiliser ATOM.
      'generated_at' => (new \DateTimeImmutable())->format(\DateTimeInterface::ATOM),
    ];

    $response = new ResourceResponse($stats, 200);
    $response->addCacheableDependency(
      \Drupal\Core\Cache\CacheableMetadata::createFromRenderArray([
        '#cache' => [
          'tags' => ['node_list:article', 'user_list'],
          'max-age' => 3600,
        ],
      ])
    );

    return $response;
  }

  private function countPublishedNodes(string $type): int {
    return (int) \Drupal::entityQuery('node')
      ->condition('type', $type)
      ->condition('status', 1)
      ->accessCheck(FALSE)
      ->count()
      ->execute();
  }

  private function countActiveUsers(): int {
    return (int) \Drupal::entityQuery('user')
      ->condition('status', 1)
      ->accessCheck(FALSE)
      ->count()
      ->execute();
  }
}
```

**Appeler :** `GET /api/v1/stats?_format=json` (le `_format` est obligatoire pour le module REST core,
contrairement à JSON:API). Pour OAuth2 : ajouter `-H "Authorization: Bearer TOKEN"`.

**Activer la ressource REST :**
```bash
# Via l'UI : /admin/config/services/rest (nécessite drupal/restui)
# OU via config YAML :

# config/install/rest.resource.mon_module_statistiques.yml
id: mon_module_statistiques
plugin_id: mon_module_statistiques
granularity: resource
configuration:
  methods:
    - GET
  formats:
    - json
  authentication:
    - basic_auth
    - cookie
    - oauth2
```
