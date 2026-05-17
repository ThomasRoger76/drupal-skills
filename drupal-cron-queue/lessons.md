# Leçons — drupal-cron-queue

Incidents cron/queue découverts en production. Mis à jour après chaque résolution.

---

## Comment ajouter une leçon

Après chaque incident lié au cron ou aux queues :
1. Identifier si le skill aurait pu prévenir l'erreur
2. Ajouter une entrée avec symptôme, cause, correction, prévention
3. Mettre à jour CHANGELOG.md

---

### 2026-05-16 — hook_cron() timeout — autres modules ne tournent pas

- **Symptôme :** Les emails de notification Drupal ne partent plus, la mise à jour des modules est bloquée
- **Cause :** Un `hook_cron()` lourd (import API de 2 minutes) bloque tout le cron
- **Correct :** Déplacer le traitement lourd dans une Queue. `hook_cron()` doit terminer en < 5 secondes
- **Prévention :** Règle absolue : `hook_cron()` = mettre des items en queue, pas traiter

### 2026-05-16 — QueueWorker `processItem()` sans exception — items perdus silencieusement

- **Symptôme :** Des items disparaissent de la queue sans être traités ni logués
- **Cause :** `processItem()` catchait toutes les exceptions et retournait sans les relancer → item considéré comme traité avec succès
- **Correct :** Ne catcher que les exceptions gérables. Toujours relancer les exceptions inattendues → Drupal remet en queue
- **Prévention :** Pattern : `catch (\Specific\Exception $e) { handle } catch (\Exception $e) { log; throw $e; }`

### 2026-05-16 — SuspendQueueException non utilisé — API externe down, queue entière traitée en erreur

- **Symptôme :** 500 items en erreur en 5 minutes, logs remplis, API externe était temporairement down
- **Cause :** Chaque item relançait une exception individuelle → retry × 500 items
- **Correct :** Détecter `ConnectException` → lancer `SuspendQueueException` → toute la queue suspendue jusqu'au prochain run
- **Prévention :** Toujours distinguer : erreur d'item (Exception normale) vs erreur de service (SuspendQueueException)

### 2026-05-16 — Batch terminé trop tôt — `$context['finished'] = 1` dans le premier chunk

- **Symptôme :** Le batch s'arrête après 50 items alors qu'il devrait en traiter 5000
- **Cause :** `$context['finished'] = 1` mis en dur au lieu d'être calculé `$processed / $total`
- **Correct :** `$context['finished'] = min(1.0, $context['sandbox']['progress'] / $total)`
- **Prévention :** Toujours calculer `finished` dynamiquement. Tester avec un dataset de 200+ items.

### 2026-05-16 — Cron non configuré en production — site jamais nettoyé

- **Symptôme :** Table watchdog de 2Go après 6 mois, sessions expirées jamais purgées
- **Cause :** Aucun crontab configuré sur le serveur de production
- **Correct :** Configurer le crontab : `*/15 * * * * drush cron --quiet`
- **Prévention :** Checklist de mise en production inclure "Vérifier crontab"

### 2026-05-16 — drush queue:run interrompu si SSH déconnecté

- **Symptôme :** `drush queue:run` s'arrête après 5 minutes en production lors d'un déploiement manuel
- **Cause :** Le processus Drush est enfant de la session SSH — il meurt quand SSH se ferme
- **Correct :** `nohup drush queue:run mon_queue --time-limit=3600 > /tmp/queue.log 2>&1 &`
- **Prévention :** Toujours utiliser `nohup` + `&` pour les commandes longues en production

### 2026-05-16 — Queue croissante — items jamais consommés

- **Symptôme :** `drush queue:list` montre des milliers d'items qui s'accumulent sans jamais diminuer
- **Cause :** `cron: {"time": 60}` manquant dans l'annotation `@QueueWorker` → le cron ne consomme pas la queue
- **Correct :** Ajouter `cron = {"time" = 60}` à l'annotation + `drush cron` pour tester
- **Prévention :** Après création d'un QueueWorker : `drush cron && drush queue:list` — vérifier que les items diminuent

### 2026-05-16 — Batch timeout HTTP — `max_execution_time` trop court

- **Symptôme :** Le batch s'arrête avec "Maximum execution time" après 60 secondes
- **Cause :** `max_execution_time = 60` dans `php.ini` — trop court pour des batchs lourds
- **Correct :** Lancer via Drush : `drush php:script mon_batch.php` → pas de timeout. OU `set_time_limit(0)` dans la première opération batch.
- **Prévention :** Batchs lourds = `drush_backend_batch_process()` depuis une commande Drush custom
