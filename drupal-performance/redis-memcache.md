---
name: drupal-performance — redis et memcache
description: Configuration de Redis et Memcache comme backends de cache Drupal - installation, settings.php, choix des bins, monitoring, et comparaison.
---

# Redis & Memcache — Backends de Cache

## Redis vs Memcache — Comparatif

| Critère | Redis | Memcache |
|---------|-------|----------|
| Persistance | ✅ (RDB/AOF optionnel) | ❌ (volatile) |
| Structures de données | ✅ (strings, lists, sets, hashes) | ❌ (strings uniquement) |
| Clustering | ✅ Redis Cluster | ✅ limité |
| Module Drupal | `drupal/redis` | `drupal/memcache` |
| Extension PHP | `phpredis` ou `predis` | `php-memcached` |
| Adoption terrain FR | ⚠️ moins courant | ✅ très répandu (OVH, Hetzner) |
| Recommandé | ✅ nouveaux projets | ✅ si fourni par l'hébergeur |

**Recommandation :**
- **Memcache** : choix dominant en agence française — disponible nativement sur la plupart des hébergeurs mutualisés et VPS (OVH, Hetzner, Infomaniak). Configuration simple, zéro friction.
- **Redis** : préférable quand disponible (meilleur monitoring, persistance optionnelle, support multi-site). À privilégier sur les nouveaux projets avec contrôle total de l'infra.
- **APCu** : toujours en complément (bins `bootstrap` et `discovery`) sur les deux configurations.

---

## Redis — Installation et Configuration

### Installation

```bash
# Module Drupal
composer require drupal/redis

# Extension PHP (dans le Dockerfile)
RUN pecl install redis && docker-php-ext-enable redis

# Service Redis dans docker-compose.yml
redis:
  image: redis:7-alpine
  ports:
    - "6379:6379"
  command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
  volumes:
    - redis_data:/data   # Persistance optionnelle

volumes:
  redis_data:
```

### Configuration settings.php

```php
// settings.php — configuration Redis comme backend de cache

// 1. Activer l'autoloader (avant le cache)
$settings['class_loader_auto_detect'] = FALSE;
require_once DRUPAL_ROOT . '/../vendor/autoload.php';

// 2. Backend Redis pour tous les bins
$settings['cache']['default'] = 'cache.backend.redis';

// 3. Ou : backends sélectifs par bin
$settings['cache']['bins']['default'] = 'cache.backend.redis';
$settings['cache']['bins']['render'] = 'cache.backend.redis';
$settings['cache']['bins']['page'] = 'cache.backend.redis';
$settings['cache']['bins']['dynamic_page_cache'] = 'cache.backend.redis';
// Le cache bootstrap reste souvent en DB pour la stabilité
$settings['cache']['bins']['bootstrap'] = 'cache.backend.chainedfast';
$settings['cache']['bins']['discovery'] = 'cache.backend.chainedfast';

// 4. Connexion Redis
$settings['redis.connection']['interface'] = 'PhpRedis';  // ou 'Predis'
$settings['redis.connection']['host'] = getenv('REDIS_HOST') ?: 'redis';
$settings['redis.connection']['port'] = (int) (getenv('REDIS_PORT') ?: 6379);
$settings['redis.connection']['base'] = 0;  // Database Redis (0-15)

// Avec authentification (Redis 6+)
$settings['redis.connection']['password'] = getenv('REDIS_PASSWORD') ?: '';

// Préfixe pour distinguer plusieurs sites sur le même Redis
$settings['redis.connection']['prefix'] = 'drupal_monsite_';
// OU dériver le préfixe de l'environnement.
// ⚠️ getenv() PHP ne prend PAS de valeur par défaut en 2e argument (contrairement à
//    la fonction env() de Symfony/Laravel) — utiliser l'opérateur ?: à la place.
$settings['cache_prefix']['default'] = (getenv('APP_ENV') ?: 'prod') . '_';

// 5. Configuration avancée du module Redis
// Options de sérialisation (via services.yml ou paramètres de connexion)
// Voir la documentation du module drupal/redis pour les options avancées
// Les paramètres varient selon la version du module — préférer la config UI
```

### Vérifier la connexion Redis

```bash
# Test depuis PHP
drush php:eval "
\$redis = \Drupal::service('redis.factory')->getClient();
echo \$redis->ping() ? 'Redis: OK' : 'Redis: FAIL';
echo PHP_EOL;
echo 'Mémoire utilisée: ' . \$redis->info()['used_memory_human'] . PHP_EOL;
"

# Test direct Redis CLI
docker compose exec redis redis-cli ping
docker compose exec redis redis-cli info memory | grep used_memory_human

# Lister les clés Drupal dans Redis
docker compose exec redis redis-cli keys "drupal_monsite_*" | head -20

# Nombre de clés
docker compose exec redis redis-cli dbsize
```

---

## Memcache — Installation et Configuration

### Installation

```bash
# Module Drupal
composer require drupal/memcache

# Extension PHP
RUN apt-get install -y libmemcached-dev && \
    pecl install memcached && \
    docker-php-ext-enable memcached

# Service dans docker-compose.yml
memcached:
  image: memcached:1.6-alpine
  command: ["-m", "256", "-c", "1024"]  # 256MB, 1024 connexions max
  ports:
    - "11211:11211"
```

### Configuration settings.php

```php
// settings.php — Memcache

// Backend pour tous les bins
$settings['cache']['default'] = 'cache.backend.memcache';

// Connexion Memcache
$settings['memcache']['servers'] = [
  'memcached:11211' => 'default',   // Format: 'host:port' => 'cluster_name'
];

// Multi-serveurs (clustering)
$settings['memcache']['servers'] = [
  'memcached1:11211' => 'default',
  'memcached2:11211' => 'default',
];

// SASL authentication (si nécessaire)
$settings['memcache']['options'] = [
  Memcached::OPT_BINARY_PROTOCOL => TRUE,
  Memcached::OPT_COMPRESSION => TRUE,
];

$settings['memcache']['sasl'] = [
  'username' => 'user',
  'password' => 'pass',
];
```

---

## APCu — Cache User Space PHP

```bash
# Installation de l'extension APCu
RUN pecl install apcu && docker-php-ext-enable apcu

# Module Drupal (sérialiseur rapide)
composer require drupal/apcu_serializer
```

```php
// settings.php — APCu pour le bootstrap (ultra-rapide)
$settings['cache']['bins']['bootstrap'] = 'cache.backend.apcu';
$settings['cache']['bins']['discovery'] = 'cache.backend.apcu';

// APCu est local à un processus PHP — pas partagé entre workers
// Idéal pour les données statiques du bootstrap Drupal
```

```ini
# php.ini — Configuration APCu
apc.enabled = 1
apc.shm_size = 64M     # Mémoire partagée (64-128MB recommandé)
apc.ttl = 7200         # TTL par défaut 2 heures
apc.gc_ttl = 3600
```

---

## Monitoring et Maintenance

### Monitoring Redis

```bash
# Stats en temps réel
docker compose exec redis redis-cli monitor | head -50  # ⚠️ Très verbeux

# Stats d'utilisation
docker compose exec redis redis-cli info stats

# Mémoire utilisée vs limite
docker compose exec redis redis-cli info memory | grep -E "used_memory_human|maxmemory_human|mem_fragmentation_ratio"

# Nombre de clés par base
docker compose exec redis redis-cli info keyspace

# Hits vs Misses (cache effectiveness)
docker compose exec redis redis-cli info stats | grep -E "keyspace_hits|keyspace_misses"
# hit_rate = hits / (hits + misses) → viser > 80%
```

### Vider le Cache Redis Drupal

```bash
# Depuis Drush (vide les bins Drupal dans Redis)
drush cr

# Vider un bin spécifique
drush php:eval "\Drupal::cache('render')->deleteAll();"

# Vider TOUT Redis (tous les bins + autres données)
docker compose exec redis redis-cli flushdb   # ⚠️ Efface la DB courante
docker compose exec redis redis-cli flushall  # ❌ Efface TOUTES les DBs
```

### Politique d'éviction Redis

```
maxmemory-policy allkeys-lru  ← Recommandé pour Drupal cache
  → Expire les clés les moins récemment utilisées quand la mémoire est pleine
  → Drupal peut toujours recalculer les valeurs évincées

maxmemory-policy noeviction   ← Bloquer les nouvelles écritures si RAM pleine
  → NE PAS utiliser pour le cache — provoque des erreurs Redis

maxmemory-policy volatile-lru ← Expire uniquement les clés avec TTL
  → Adapté si Drupal utilise TTL sur toutes ses clés
```

---

## Troubleshooting

```bash
# Redis non accessible → vérifier la connexion
docker compose exec php php -r "
\$r = new Redis();
\$r->connect('redis', 6379);
echo \$r->ping() ? 'OK' : 'FAIL';
"

# Module Redis non chargé → vérifier l'autoloader
drush php:eval "echo class_exists('Drupal\redis\Cache\PhpRedis') ? 'OK' : 'Module non chargé';"

# Trop de miss → vérifier que les bins sont correctement configurés
drush php:eval "var_dump(\Drupal::service('settings')->get('cache'));"

# Fragmentation mémoire Redis > 1.5 → restart recommandé
docker compose exec redis redis-cli info memory | grep mem_fragmentation_ratio
docker compose restart redis   # Déframente la mémoire
```
