---
name: drupal-performance — profiling
description: Profiler les performances Drupal avec Devel/Kint, XHProf, Blackfire.io, WebPageTest, et analyser les slow queries. Identifier les bottlenecks PHP et DB.
---

# Profiling Drupal — Outils et Méthodes

## Devel — Debug Rapide en Développement

```bash
composer require drupal/devel drupal/devel_kint_extras
drush en devel kint -y
```

### Afficher les Requêtes SQL

```php
// Activer le query log (settings.local.php)
$settings['devel.settings']['query_display'] = TRUE;
$settings['devel.settings']['query_sort'] = 'query';

// OU dans du code temporaire
\Drupal::state()->set('devel.query_display', TRUE);
```

### Fonctions de Debug Devel

```php
// kint() — dump structuré avec Kint (meilleur que var_dump)
kint($node);
kint($view->result);

// dpm() — debug dans les messages Drupal
dpm($variables, 'Variables disponibles');

// dvm() — debug en utilisant var_export
dvm($array, 'Mon tableau');

// dpq() — afficher une query Views
dpq($view->query);

// dd() — debug dans un fichier log (pas dans la page HTML)
dd($data, 'Debug silencieux');

// dsm() — message Drupal simple
dsm('Valeur de X : ' . $x);

// Timer (mesure manuelle)
timer_start('mon_timer');
// ... code à mesurer ...
$elapsed = timer_read('mon_timer');
dpm($elapsed . 'ms', 'Temps de traitement');
```

---

## Xdebug — Profiling PHP

```ini
# php.ini — activer le profiling Xdebug
[xdebug]
zend_extension = xdebug.so
xdebug.mode = profile
xdebug.start_with_request = trigger    # Démarrer sur demande (pas toujours)
xdebug.output_dir = /tmp/xdebug       # Fichiers cachegrind
xdebug.profiler_output_name = cachegrind.out.%p   # Nom du fichier
```

```bash
# Déclencher le profiling sur une requête
curl "https://mon-site.com/node/1?XDEBUG_TRIGGER=1"

# Ouvrir le fichier cachegrind dans KCacheGrind (Linux) ou qcachegrind (Mac)
kcachegrind /tmp/xdebug/cachegrind.out.12345

# PHP CLI — profiler un script Drush
php -d xdebug.mode=profile \
    -d xdebug.output_dir=/tmp/xdebug \
    vendor/bin/drush migrate:import articles
```

---

## XHProf — Profiling Léger en Production

```bash
# Installation extension PHP
RUN pecl install xhprof && docker-php-ext-enable xhprof

# Module Drupal
composer require drupal/xhprof
drush en xhprof -y
```

```php
// Configuration xhprof module
// /admin/config/development/xhprof

// OU manuellement dans le code
// (généralement géré par le module)
xhprof_enable(XHPROF_FLAGS_CPU | XHPROF_FLAGS_MEMORY);
// ... code à profiler ...
$xhprof_data = xhprof_disable();

// Analyser les données
$xhprof_namespace = 'drupal';
$xhprof_runs = new XHProfRuns_Default('/tmp/xhprof');
$run_id = $xhprof_runs->save_run($xhprof_data, $xhprof_namespace);
echo "http://localhost/xhprof/xhprof_html/index.php?run={$run_id}&source={$xhprof_namespace}";
```

---

## Blackfire.io — Profiling Professionnel

```bash
# Installation dans le container PHP
RUN curl https://packages.blackfire.io/gpg.key | gpg --dearmor > /etc/apt/trusted.gpg.d/blackfire.gpg
RUN echo "deb https://packages.blackfire.io/debian any main" > /etc/apt/sources.list.d/blackfire.list
RUN apt-get update && apt-get install -y blackfire
```

```yaml
# docker-compose.yml — agent Blackfire
blackfire:
  image: blackfire/blackfire:2
  ports:
    - "8307:8307"
  environment:
    BLACKFIRE_SERVER_ID: ${BLACKFIRE_SERVER_ID}
    BLACKFIRE_SERVER_TOKEN: ${BLACKFIRE_SERVER_TOKEN}
    BLACKFIRE_CLIENT_ID: ${BLACKFIRE_CLIENT_ID}
    BLACKFIRE_CLIENT_TOKEN: ${BLACKFIRE_CLIENT_TOKEN}
```

```bash
# Profiler une page
blackfire curl https://mon-site.com/node/1

# Profiler une commande Drush
blackfire run drush migrate:import --all

# Comparer deux profiles (avant/après optimisation)
blackfire compare PROFILE_ID_1 PROFILE_ID_2
```

### Assertions Blackfire (CI/CD)

```yaml
# .blackfire.yaml — seuils automatiques
scenarios:
  Home:
    - visit: /
      name: "Page d'accueil"
      assertions:
        - "metrics.sql.queries.count < 20"
        - "main.wall_time < 500ms"
        - "main.memory < 50mb"

  Node:
    - visit: /node/1
      name: "Article"
      assertions:
        - "metrics.sql.queries.count < 30"
        - "main.wall_time < 300ms"
```

---

## WebPageTest & Lighthouse

```bash
# Lighthouse CLI
npm install -g lighthouse

# Audit complet
lighthouse https://mon-site.com \
  --output html \
  --output-path rapport-lighthouse.html \
  --chrome-flags="--headless"

# Avec throttling réseau (3G)
lighthouse https://mon-site.com \
  --throttling-method=simulate \
  --throttling.rttMs=150 \
  --throttling.throughputKbps=1638.4

# Focus sur les Core Web Vitals
lighthouse https://mon-site.com --only-categories=performance
```

---

## Analyser les Slow Queries

```bash
# Activer le slow query log MariaDB
docker compose exec database mysql -u root -proot -e "
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 1;
SET GLOBAL log_queries_not_using_indexes = 1;
"

# Lire le slow query log
docker compose exec database tail -100 /var/lib/mysql/slow-queries.log

# Analyzer les requêtes avec mysqldumpslow
mysqldumpslow -s t -t 10 slow-queries.log    # Top 10 par temps total
mysqldumpslow -s c -t 10 slow-queries.log    # Top 10 par nombre d'occurrences

# EXPLAIN d'une requête Drupal lente
drush sql-query "
EXPLAIN SELECT nfd.nid, nfd.title
FROM node_field_data nfd
LEFT JOIN node__field_tags nft ON nfd.nid = nft.entity_id
WHERE nfd.status = 1
AND nfd.type = 'article'
AND nfd.langcode = 'fr'
ORDER BY nfd.created DESC
LIMIT 10
"
```

---

## Métriques Clés à Surveiller

| Métrique | Cible | Outil |
|---------|-------|-------|
| TTFB (Time To First Byte) | < 200ms | WebPageTest, curl |
| SQL queries par page | < 20 (anonyme) | Devel query log |
| Mémoire PHP par requête | < 64MB | Blackfire |
| Hit rate OPcache | > 90% | opcache_get_status() |
| Hit rate Redis | > 80% | redis-cli info stats |
| LCP | < 2.5s | Lighthouse |
| CLS | < 0.1 | Lighthouse |
| INP | < 200ms | Lighthouse |

```bash
# Quick benchmark TTFB
curl -o /dev/null -s -w "%{time_starttransfer}s\n" https://mon-site.com/
curl -o /dev/null -s -w "%{time_starttransfer}s\n" https://mon-site.com/  # 2ème (cache)

# Comparer MISS vs HIT
for i in 1 2 3; do
  curl -o /dev/null -s -w "Request $i: %{time_starttransfer}s\n" https://mon-site.com/
done
```
