---
name: drupal-cron-queue — cron management
description: Configurer et gérer le cron Drupal - hook_cron(), drush cron, crontab système, Ultimate Cron, et monitoring.
---

# Cron Management — Référence Complète

## hook_cron() — Tâches Périodiques Légères

```php
<?php

/**
 * Implements hook_cron().
 *
 * S'exécute à chaque run de cron. Garder léger (< 5 secondes).
 * Pour les tâches lourdes → Queue API.
 */
function mon_module_cron(): void {
  // Nettoyer les vieilles entrées (exemple : purger les logs > 30 jours)
  $cutoff = \Drupal::time()->getRequestTime() - 30 * 24 * 3600;

  $deleted = \Drupal::database()->delete('mon_module_events')
    ->condition('created', $cutoff, '<')
    ->execute();

  if ($deleted > 0) {
    \Drupal::logger('mon_module')->info(
      '@count entrées nettoyées par cron.',
      ['@count' => $deleted]
    );
  }

  // Exemple : vérifier un flux RSS et mettre en queue les nouveaux items
  $last_check = \Drupal::state()->get('mon_module.last_rss_check', 0);
  $interval = 3600; // vérifier toutes les heures

  if (\Drupal::time()->getRequestTime() - $last_check >= $interval) {
    // Mettre en queue le traitement lourd
    $queue = \Drupal::queue('mon_module_rss_import');
    $queue->createItem(['url' => 'https://example.com/feed.rss']);
    \Drupal::state()->set('mon_module.last_rss_check', \Drupal::time()->getRequestTime());
  }
}
```

---

## Configurer et Déclencher le Cron

```bash
# Lancer le cron manuellement
drush cron

# Lancer le cron pour un module spécifique
drush php:eval "
\$module_handler = \Drupal::moduleHandler();
if (\$module_handler->implementsHook('mon_module', 'cron')) {
  \$module_handler->invoke('mon_module', 'cron');
  echo 'Cron mon_module exécuté.';
}
"

# Vérifier le statut du cron
drush core:requirements | grep -i cron
# OU via l'UI : /admin/reports/status

# Voir quand le cron a tourné pour la dernière fois
drush php:eval "echo date('Y-m-d H:i:s', \Drupal::state()->get('system.cron_last'));"
```

---

## Crontab Système — Configuration

```bash
# Crontab recommandé pour Drupal (toutes les 15 minutes)
# Ouvrir le crontab
crontab -e

# Ajouter ces lignes (adapter les chemins)
*/15 * * * * cd /var/www/mon-site && vendor/bin/drush cron --quiet 2>&1 | logger -t drupal-cron

# Avec DDEV
*/15 * * * * cd /var/www/mon-site && ddev drush cron --quiet 2>&1 | logger -t drupal-cron

# Avec Docker Compose
*/15 * * * * cd /var/www/mon-site && docker compose exec -T php vendor/bin/drush cron --quiet

# Vérifier les logs du cron
grep drupal-cron /var/log/syslog | tail -20
```

---

## Ultimate Cron — Timing Précis par Module

```bash
composer require drupal/ultimate_cron
drush en ultimate_cron -y
```

```yaml
# Configuration Ultimate Cron par module
# /admin/config/system/cron → configurer chaque cron job individuellement

# config/install/ultimate_cron.job.mon_module_cron.yml
langcode: fr
status: true
id: mon_module_cron
title: 'Mon Module — Traitement RSS'
module: mon_module
name: cron
scheduler:
  id: crontab
  configuration:
    crontab: '*/30 * * * *'    # Toutes les 30 minutes
launcher:
  id: serial                    # Pas de lancement parallèle
  configuration:
    timeouts:
      timeout: 120              # Max 2 minutes
      lock_timeout: 120
logger:
  id: database
  configuration:
    logs_per_job: 20
```

---

## Monitoring du Cron

```bash
# Voir tous les jobs cron enregistrés
drush php:eval "
\$cron = \Drupal::service('ultimate_cron.discovery');
foreach (\$cron->getJobHooks() as \$module => \$jobs) {
  echo \$module . ': ' . count(\$jobs) . ' job(s)' . PHP_EOL;
}
"

# Avec Ultimate Cron — voir l'état de chaque job
drush php:eval "
\$jobs = \Drupal::entityTypeManager()->getStorage('ultimate_cron_job')->loadMultiple();
foreach (\$jobs as \$job) {
  \$log = \$job->loadLatestLog();
  echo \$job->id() . ': last run=' . (\$log ? date('Y-m-d H:i', \$log->getStartTime()) : 'never') . PHP_EOL;
}
"

# Déclencher un job spécifique Ultimate Cron
drush ultimate-cron:run mon_module_cron
```
