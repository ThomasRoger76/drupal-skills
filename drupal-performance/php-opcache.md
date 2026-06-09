---
name: drupal-performance — PHP OPcache et JIT
description: Configuration OPcache, JIT PHP 8.3, APCu, et preload pour Drupal - valeurs optimales, settings dev vs prod, et monitoring des performances PHP.
---

# PHP OPcache, JIT & APCu — Référence Complète

## OPcache — Configuration Optimale

```ini
# /usr/local/etc/php/conf.d/opcache.ini (Docker)

; ── Activation ────────────────────────────────────────────────────────────
opcache.enable = 1
opcache.enable_cli = 0        ; Désactiver en CLI (Drush) sauf si nécessaire

; ── Mémoire ───────────────────────────────────────────────────────────────
opcache.memory_consumption = 256       ; MB — 128 minimum, 256 recommandé
opcache.interned_strings_buffer = 16   ; MB pour les chaînes internalisées
opcache.max_accelerated_files = 20000  ; Nombre max de fichiers PHP cachés
                                        ; drush php:eval "echo count(get_included_files());"
                                        ; → valeur de référence

; ── Revalidation ──────────────────────────────────────────────────────────
; PRODUCTION : validate_timestamps = 0 (ne pas vérifier les modifications)
opcache.validate_timestamps = 0        ; ← OBLIGATOIRE en production
opcache.revalidate_freq = 0            ; Ignoré si validate_timestamps = 0
; → Après déploiement : drush cr pour invalider OPcache

; DÉVELOPPEMENT : revalider fréquemment
; opcache.validate_timestamps = 1
; opcache.revalidate_freq = 2          ; Vérifier toutes les 2 secondes

; ── Stabilité ─────────────────────────────────────────────────────────────
opcache.fast_shutdown = 1
opcache.consistency_checks = 0        ; Désactiver en prod (performance)
opcache.max_wasted_percentage = 10    ; Restart si 10% gaspillé

; ── Accélération supplémentaire ───────────────────────────────────────────
opcache.save_comments = 1             ; OBLIGATOIRE pour Drupal (annotations)
opcache.enable_file_override = 0
```

---

## Différences Dev vs Prod

```ini
# DÉVELOPPEMENT
opcache.validate_timestamps = 1    ; Vérifier à chaque requête
opcache.revalidate_freq = 0        ; Immédiatement
opcache.save_comments = 1          ; Annotations Drupal
opcache.memory_consumption = 128   ; Moins critique

# PRODUCTION
opcache.validate_timestamps = 0    ; Jamais vérifier → drush cr après déploiement
opcache.save_comments = 1          ; TOUJOURS — Drupal les utilise pour les plugins
opcache.memory_consumption = 256   ; Plus de mémoire pour les fichiers
opcache.max_accelerated_files = 20000
```

**⚠️ Attention :** `save_comments = 0` CASSE Drupal en production — les annotations de plugins ne sont plus lues.

---

## JIT PHP 8.3 — Quand l'Activer

```ini
; JIT (Just-In-Time compiler) — disponible depuis PHP 8.0
; Pour Drupal : gain marginal sur le code PHP dynamique
; Meilleur gain : scripts de migration, traitement batch, workers Drupal

opcache.jit = 1255               ; tracing mode — recommandé
opcache.jit_buffer_size = 128M   ; Mémoire JIT

; Valeurs de jit :
; 0     = désactivé
; 1205  = function mode (compile les fonctions fréquentes)
; 1255  = tracing mode (analyse les patterns d'exécution — recommandé)
; on    = alias de 1255
; tracing = alias de 1255
```

**Quand JIT apporte le plus :**
```bash
# Migration Drupal — CPU intensif
drush migrate:import --all   # +20-40% avec JIT

# Génération de sitemap — boucle sur beaucoup d'entités
# Traitement batch d'images (resize, WebP conversion)
# Import CSV de millions de lignes

# Quand JIT N'aide pas (ou très peu) :
# - Requêtes Drupal normales (I/O bound, pas CPU bound)
# - Pages avec beaucoup de requêtes DB
# - Sites avec Page Cache actif (PHP n'est pas appelé)
```

**Benchmarker avant d'activer :**
```bash
# Benchmark sans JIT
docker compose exec php php -d opcache.jit=0 -d opcache.jit_buffer_size=0 \
  vendor/bin/drush php:eval "
  \$start = microtime(TRUE);
  for (\$i = 0; \$i < 1000; \$i++) {
    node_access_rebuild_recursive();  // Opération CPU intensive
  }
  echo microtime(TRUE) - \$start;
  "

# Benchmark avec JIT
docker compose exec php php -d opcache.jit=1255 -d opcache.jit_buffer_size=128M \
  vendor/bin/drush php:eval "..."
```

---

## Monitoring OPcache

```php
// Page de monitoring OPcache (dev uniquement)
// Vérifier l'utilisation actuelle
drush php:eval "
\$status = opcache_get_status(false);
\$config = opcache_get_configuration();

\$memory = \$status['memory_usage'];
\$total = \$memory['used_memory'] + \$memory['free_memory'] + \$memory['wasted_memory'];

echo 'Mémoire utilisée : ' . round(\$memory['used_memory'] / 1024 / 1024, 1) . 'MB' . PHP_EOL;
echo 'Mémoire libre : ' . round(\$memory['free_memory'] / 1024 / 1024, 1) . 'MB' . PHP_EOL;
echo 'Mémoire gaspillée : ' . round(\$memory['wasted_memory'] / 1024 / 1024, 1) . 'MB' . PHP_EOL;
echo 'Fichiers mis en cache : ' . \$status['opcache_statistics']['num_cached_scripts'] . PHP_EOL;
echo 'Hit rate : ' . round(\$status['opcache_statistics']['opcache_hit_rate'], 2) . '%' . PHP_EOL;
"
```

```bash
# Invalider OPcache après déploiement
# Option 1 : restart PHP (le plus fiable)
docker compose restart php

# Option 2 : depuis PHP (si opcache_reset est disponible)
drush php:eval "opcache_reset();"

# Option 3 : via drush cr (invalide aussi le cache Drupal)
drush cr

# Option 4 : déploiement automatique (Makefile)
deploy:
	git pull
	docker compose exec php composer install --no-dev --optimize-autoloader
	docker compose exec php drush deploy
	docker compose restart php  # Invalide OPcache
```

---

## OPcache Preload (PHP 7.4+)

> **⚠️ Drupal ne supporte PAS officiellement `opcache.preload`.** Drupal core n'a pas de
> fichier de preload fourni, et un preload naïf casse PHP-FPM au démarrage. Ne PAS activer
> sans benchmark ni test de charge. Sur la majorité des projets, OPcache classique + APCu
> suffisent largement. Voir l'issue Drupal #3055735.

**Pourquoi un preload naïf est dangereux :**

```php
// ❌ NE JAMAIS FAIRE — compiler tout vendor/ brutalement
// → fatal errors au boot PHP-FPM : "Cannot declare class ... already in use",
//   classes avec dépendances non résolvables, traits/interfaces dans le désordre,
//   code conditionnel (if class_exists) compilé hors contexte.
$iterator = new RecursiveIteratorIterator(new RecursiveDirectoryIterator('vendor'));
foreach ($iterator as $file) {
  if ($file->getExtension() === 'php') {
    opcache_compile_file($file->getPathname());  // ← casse le démarrage
  }
}
```

**Approche correcte (si vraiment nécessaire) :** ne précharger qu'une liste curée et stable
de classes "feuilles" (sans dépendances non résolvables), via `opcache_compile_file()`,
en testant le redémarrage PHP-FPM après chaque ajout. Tolérer les erreurs avec un `try/catch`.

```php
// opcache-preload.php — liste curée, tolérante aux erreurs
<?php
$classmap = require __DIR__ . '/vendor/composer/autoload_classmap.php';
foreach ($classmap as $class => $path) {
  // Cibler uniquement le code applicatif stable, jamais tout vendor/
  if (!str_contains($path, '/vendor/symfony/')) {
    continue;
  }
  try {
    opcache_compile_file($path);
  }
  catch (\Throwable $e) {
    // Une classe non préchargeable ne doit jamais casser le boot
  }
}
```

```ini
# php.ini — activer le preload (uniquement après validation)
opcache.preload = /var/www/html/opcache-preload.php
# preload_user requis SEULEMENT si PHP-FPM démarre en root (sinon optionnel) :
opcache.preload_user = www-data
```

---

## APCu — User Cache PHP

```bash
# Installation
RUN pecl install apcu && docker-php-ext-enable apcu
```

```ini
# php.ini — APCu
extension = apcu.so
apc.enabled = 1
apc.shm_size = 64M      ; Mémoire partagée
apc.ttl = 7200          ; TTL par défaut (2h)
apc.gc_ttl = 3600
apc.enable_cli = 0      ; Désactiver en CLI
```

```php
// Utilisation directe d'APCu dans le code Drupal
$cid = 'mon_module:data:' . $id;

if (!$data = apcu_fetch($cid)) {
  $data = $this->computeExpensiveData($id);
  apcu_store($cid, $data, 3600);  // TTL 1 heure
}

// Ou via le module drupal/apcu_serializer
// → accélérer le cache Drupal bootstrap avec APCu comme sérialiseur
```

---

## Checklist PHP Performance

```
[ ] OPcache activé (opcache.enable = 1)
[ ] validate_timestamps = 0 en production
[ ] save_comments = 1 (obligatoire Drupal)
[ ] memory_consumption >= 128MB
[ ] max_accelerated_files >= 10000
[ ] JIT activé si usage intensif CPU (migrations, batch)
[ ] Preload : NON par défaut — Drupal ne le supporte pas officiellement (cf. section dédiée)
[ ] APCu pour le cache bootstrap Drupal
[ ] drush cr après chaque déploiement
[ ] Monitoring : hit_rate > 90% = bien configuré
```
