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

# Avec Docker Compose (pattern recommandé — -T désactive l'allocation TTY, indispensable en cron)
*/15 * * * * cd /var/www/mon-site && docker compose exec -T php drush cron --quiet 2>&1 | logger -t drupal-cron

# Vérifier les logs du cron
grep drupal-cron /var/log/syslog | tail -20
```

---

## Lock Service — Éviter les Chevauchements de Cron Long

Si un `hook_cron()` peut dépasser l'intervalle du crontab (ex. job de 20 min lancé toutes les 15 min),
deux runs se chevauchent et corrompent l'état. Le `lock` service garantit l'exclusion mutuelle.

```php
function mon_module_cron(): void {
  $lock = \Drupal::lock();

  // Tenter d'acquérir un verrou de 1800s. Si déjà pris, on sort (run précédent en cours).
  if (!$lock->acquire('mon_module_import_lourd', 1800)) {
    \Drupal::logger('mon_module')->notice('Import déjà en cours, run ignoré.');
    return;
  }

  try {
    // ... traitement long (idéalement déléguer à une Queue ; le lock couvre le edge case) ...
  }
  finally {
    $lock->release('mon_module_import_lourd');
  }
}
```

---

## Ultimate Cron — Timing Précis par Module

> ⚠️ **D11 : pas de release stable** — seule la `8.x-2.0-beta1` cible `^11`.
> Pour un nouveau projet D11, préférer **plusieurs lignes crontab** appelant des commandes Drush
> ciblées (`drush queue:run`, commande custom) plutôt qu'un seul `drush cron`. Réserver Ultimate Cron
> aux sites legacy qui en dépendent déjà.

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
