---
name: drupal-cron-queue
description: Use when configuring Drupal cron (drush cron, crontab scheduling, Ultimate Cron module for precise timing), creating Queue Worker plugins (@QueueWorker / #[QueueWorker]) to process background tasks asynchronously, adding items to queues with QueueFactory (createItem, numberOfItems, deleteItem), running queues manually with drush queue:run, implementing Batch API (batch_set, operations array, finished callback) for processing thousands of items without timeout, handling queue errors with SuspendQueueException or RequeueException, monitoring queue state with drush queue:list, implementing hook_cron() for lightweight periodic tasks, testing queue workers with KernelTestBase, or diagnosing cron/queue issues in Drupal 8-11+
---

# Drupal Cron & Queue — Référence Complète

## Overview

Référentiel complet du système de traitement asynchrone et planifié Drupal 8-11+ : cron (hook_cron, drush cron, crontab), Queue API (QueueWorker plugin, processItem, retry), Batch API (milliers d'items sans timeout), et monitoring. Ces trois mécanismes couvrent 100% des besoins de traitement en arrière-plan.

## 🎯 Choisir le Bon Mécanisme

```
Tâche légère, périodique (nettoyage, emails, stats)
  → hook_cron() — simple, s'exécute à chaque cron run

Traitement lourd, asynchrone (import API, emails en masse)
  → Queue API — items traités en background, retry automatique

Traitement massif, avec progression (migration, rebuild index)
  → Batch API — barre de progression, pas de timeout, contexte persistant
```

---

## Quick Decision Table

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Exécuter du code à chaque run de cron | `hook_cron()` dans `.module` | [cron-management.md](cron-management.md) |
| Lancer le cron depuis la ligne de commande | `drush cron` | [cron-management.md](cron-management.md) |
| Configurer un crontab système | `* * * * * drush cron` | [cron-management.md](cron-management.md) |
| Cron avec timing précis par module | `drupal/ultimate_cron` | [cron-management.md](cron-management.md) |
| Vérifier l'état du cron | `drush php:eval + /admin/reports/status` | [cron-management.md](cron-management.md) |
| Traiter des tâches lourdes en arrière-plan | Queue API — `QueueWorker` plugin | [queue-workers.md](queue-workers.md) |
| Ajouter un item à traiter en queue | `\Drupal::queue('ma_queue')->createItem($data)` | [queue-workers.md](queue-workers.md) |
| Traiter une queue manuellement | `drush queue:run ma_queue` | [queue-workers.md](queue-workers.md) |
| Lister les queues et leur taille | `drush queue:list` | [queue-workers.md](queue-workers.md) |
| Retry automatique si `processItem()` échoue | Lancer une exception → Drupal remet en queue | [queue-workers.md](queue-workers.md) |
| Suspendre une queue (resource overload) | `SuspendQueueException` | [queue-workers.md](queue-workers.md) |
| Traiter des milliers d'items sans timeout | Batch API — `batch_set()` | [batch-api.md](batch-api.md) |
| Batch avec barre de progression dans l'UI | `batch_set()` depuis un Controller | [batch-api.md](batch-api.md) |
| Batch depuis Drush (CLI) | `drush_backend_batch_process()` | [batch-api.md](batch-api.md) |
| Persister des données entre les chunks batch | `$context['sandbox']` | [batch-api.md](batch-api.md) |
| Tester un QueueWorker (PHPUnit) | `KernelTestBase + plugin.manager.queue_worker` | [queue-workers.md](queue-workers.md) |
| Email en masse sans timeout | Queue API — 1 item = 1 email | [queue-workers.md](queue-workers.md) |
| Importer un CSV de 10 000 lignes | Batch API — chunk de 50 lignes | [batch-api.md](batch-api.md) |
| Rebuild du cache d'entités large | Batch API | [batch-api.md](batch-api.md) |
| Nettoyer les vieux logs watchdog | `hook_cron()` avec `time() - 604800` | [cron-management.md](cron-management.md) |
| **Monitorer la taille des queues en production** | `drush queue:list` dans un script cron + alerte si items > seuil | [queue-workers.md](queue-workers.md) |
| **Queue avec backend Redis (haute performance)** | `settings.php` → `$settings['queue_service_QUEUE_NAME'] = 'queue.reliable_queue'` + `drupal/redis` pour le cache associé | [queue-workers.md](queue-workers.md) |
| **Priorité entre plusieurs queues** | Exécuter d'abord la queue critique : `drush queue:run critique && drush queue:run secondaire` | [queue-workers.md](queue-workers.md) |
| **Cron par module avec timing précis** | Ultimate Cron → créer un job `hook_cron()` avec `rules.crontab: '*/5 * * * *'` | [cron-management.md](cron-management.md) |
| **Rejouer les items en erreur** | `RequeueException` → remet l'item en queue avec backoff exponentiel | [queue-workers.md](queue-workers.md) |
| **Queue items avec expiration (TTL)** | Stocker `created` dans le payload + vérifier `time() - $item->created < 3600` dans `processItem()` | [queue-workers.md](queue-workers.md) |
| **Batch dans un QueueWorker (chunk de 100)** | `processItem()` qui crée un sous-batch de 100 items max | [batch-api.md](batch-api.md) |
| **Tracer les items traités / en erreur** | Logger dans watchdog + `drush watchdog:show --type=mon_queue` | [queue-workers.md](queue-workers.md) |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| Traitement lourd dans `hook_cron()` | Queue API pour les tâches > 5 secondes | Cron timeout, autres modules ne tournent pas |
| `set_time_limit(0)` dans un Controller web | Batch API ou Queue | Timeout serveur PHP |
| Loop sur 10 000 entités sans Batch | Batch API avec chunk de 50 | Memory exhausted, timeout HTTP |
| Queue sans `processItem()` qui lance une exception en cas d'erreur | Toujours relancer une exception pour le retry | Items silencieusement perdus |
| Cron toutes les minutes sur un VPS partagé | Ultimate Cron avec timing adapté par opération | Surcharge serveur |
| `drush queue:run` en production sans nohup | `nohup drush queue:run & ou cron wrapper` | Process tué si SSH déconnecté |
| `$context['finished'] = TRUE` trop tôt | Calculer `$finished = $processed / $total` | Batch se termine avant la fin |

## Évolution par Version Majeure

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| `hook_cron()` | ✅ | ✅ | ✅ | ✅ |
| Queue API (core) | ✅ | ✅ | ✅ | ✅ |
| `@QueueWorker` annotation | ✅ | ✅ | ✅ | ⚠️ déprécié |
| `#[QueueWorker]` attribute | ❌ | ❌ | ✅ opt. | ✅ standard |
| Batch API | ✅ | ✅ | ✅ | ✅ |
| `drush queue:run` | ✅ | ✅ | ✅ | ✅ |
| `drush queue:list` | ✅ | ✅ | ✅ | ✅ |
| Ultimate Cron (contrib) | ✅ | ✅ | ✅ | ✅ |
| `SuspendQueueException` | ✅ | ✅ | ✅ | ✅ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Incidents cron/queue découverts en production.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions.

## See Also

- `drupal-core` — QueueInterface, QueueFactory, Batch API dans le module system
- `drupal-docker` — Cron via container, crontab dans Docker
- `drupal-testing` — Tester QueueWorker avec KernelTestBase
- `drupal-performance` — Queue pour éviter les timeouts HTTP
- `drupal-migration` — Batch API dans les migrations Drupal
