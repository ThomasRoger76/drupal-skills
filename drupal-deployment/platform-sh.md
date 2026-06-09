---
name: drupal-deployment — platform.sh
description: Déployer Drupal sur Platform.sh - .platform.app.yaml, routes.yaml, services.yaml, CLI, hooks de déploiement, et variables d'environnement.
---

# Platform.sh — Déploiement Drupal

## Structure de Configuration

```
.platform/
├── routes.yaml         ← Définition des URLs (domaines, redirections)
└── services.yaml       ← Services (MariaDB, Redis, Solr...)

.platform.app.yaml      ← Configuration de l'application
```

---

## `.platform.app.yaml` — Configuration Principale

```yaml
# .platform.app.yaml — D11 supporte php:8.3 et php:8.4 (PHP 8.3+ requis par D11)
name: app
type: php:8.3

# Variables d'environnement
variables:
  php:
    memory_limit: 512M

# Build — exécuté une fois par commit (pas de DB)
hooks:
  build: |
    set -e
    composer install --no-dev --optimize-autoloader
    # Vider les assets compilés (SCSS/JS si Webpack)

  # Deploy — exécuté après chaque déploiement (avec accès DB)
  deploy: |
    set -e
    drush -y deploy
    drush -y php:eval "drupal_flush_all_caches();"

  # Post-deploy — exécuté après que le trafic est redirigé vers la nouvelle version
  post_deploy: |
    drush cr

# Point de montage pour les fichiers Drupal (persistants entre déploiements)
mounts:
  web/sites/default/files:
    source: local
    source_path: files
  /tmp:
    source: local
    source_path: tmp
  /private:
    source: local
    source_path: private

# Relations avec les services définis dans services.yaml
relationships:
  database: "db:mysql"
  redis: "cache:redis"

# Web server configuration
web:
  locations:
    /:
      root: web
      expires: 5m
      passthru: /index.php
      allow: false
      rules:
        '\.(jpe?g|png|gif|svgz?|css|js|map|ico|bmp|eot|woff2?|otf|ttf)$':
          allow: true
        '^/robots\.txt$':
          allow: true
        '^/sitemap\.xml$':
          allow: true
    /sites/default/files:
      allow: true
      expires: 1d
      passthru: /index.php
      root: web/sites/default/files
      scripts: false
```

---

## `services.yaml` — Services Hébergés

```yaml
# .platform/services.yaml

db:
  type: mariadb:11.0
  disk: 2048        # MB

cache:
  type: redis:7.0

search:
  type: solr:9.3
  disk: 1024
  configuration:
    core_config: !archive configuration
```

---

## `routes.yaml` — Configuration des URLs

```yaml
# .platform/routes.yaml

https://{default}/:
  type: upstream
  upstream: app:http
  cache:
    enabled: true
    headers:
      - Accept
      - Accept-Language
    cookies:
      - /^SESS/        # Cookies de session Drupal
    default_ttl: 0

# Redirection www → non-www
https://www.{default}/:
  type: redirect
  to: https://{default}/
```

---

## CLI Platform.sh

```bash
# Installer la CLI
curl -fsSL https://platform.sh/cli/installer | bash

# Se connecter
platform auth:login

# Lister les projets
platform project:list

# SSH vers un environnement
platform ssh -p PROJECT_ID -e main

# Ouvrir un tunnel vers la DB (pour Sequel Pro / TablePlus)
platform tunnel:open -p PROJECT_ID -e main

# Copier DB prod → local
platform db:dump -p PROJECT_ID -e main --gzip -f prod-dump.sql.gz
gunzip -c prod-dump.sql.gz | drush sql:cli  # Importer en local (-c = stdout)

# Lancer le déploiement (force redeploy)
platform environment:redeploy -p PROJECT_ID -e main

# Voir les logs de déploiement
platform activity:list -p PROJECT_ID -e main
platform activity:get ACTIVITY_ID --log

# Variables d'environnement
platform variable:set -p PROJECT_ID -e main SECRET_KEY "valeur-secrete" --sensitive
platform variable:list -p PROJECT_ID -e main
```

---

## Variables d'Environnement Platform.sh

```php
// settings.platformsh.php — inclus automatiquement via Drupal template Platform.sh

// Connexion DB depuis les relations Platform.sh
if (isset($_ENV['PLATFORM_RELATIONSHIPS'])) {
  $relationships = json_decode(base64_decode($_ENV['PLATFORM_RELATIONSHIPS']), TRUE);

  if (isset($relationships['database'])) {
    $db = $relationships['database'][0];
    $databases['default']['default'] = [
      'driver' => 'mysql',
      'database' => $db['path'],
      'username' => $db['username'],
      'password' => $db['password'],
      'host' => $db['host'],
      'port' => $db['port'],
    ];
  }

  if (isset($relationships['redis'])) {
    $redis = $relationships['redis'][0];
    $settings['redis.connection']['host'] = $redis['host'];
    $settings['redis.connection']['port'] = $redis['port'];
    $settings['cache']['default'] = 'cache.backend.redis';
  }
}

// Hash salt depuis variable Platform.sh
if (isset($_ENV['PLATFORM_PROJECT_ENTROPY'])) {
  $settings['hash_salt'] = $_ENV['PLATFORM_PROJECT_ENTROPY'];
}
```
