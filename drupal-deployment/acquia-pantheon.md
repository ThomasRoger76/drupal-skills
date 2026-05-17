---
name: drupal-deployment — acquia et pantheon
description: Déployer Drupal sur Acquia Cloud et Pantheon - Terminus CLI, Acquia CLI, Drush aliases, sync DB/files, et pipelines automatisés.
---

# Acquia & Pantheon — Référence Complète

## Pantheon — Terminus CLI

```bash
# Installer Terminus
curl -L https://github.com/pantheon-systems/terminus/releases/latest/download/terminus.phar -o /usr/local/bin/terminus
chmod +x /usr/local/bin/terminus

# Authentification
terminus auth:login --machine-token=TOKEN

# Lister les sites
terminus site:list

# Infos d'un environnement
terminus env:info SITE.ENV

# === Déploiement ===

# Déployer dev → test
terminus env:deploy SITE.test --updatedb --note="Deploy from dev"

# Déployer test → live (production)
terminus env:deploy SITE.live --updatedb --note="Release v1.2"

# Drush sur Pantheon
terminus remote:drush SITE.live -- cr
terminus remote:drush SITE.live -- deploy

# === Gestion DB / Files ===

# Sync DB live → dev (pour tester avec données prod)
terminus env:clone-content SITE.live SITE.dev --db-only

# Sync files live → dev
terminus env:clone-content SITE.live SITE.dev --files-only

# Dump DB
terminus remote:drush SITE.live -- sql:dump --gzip --result-file=/tmp/dump.sql.gz

# === Logs ===
terminus remote:drush SITE.live -- watchdog:show --count=50

# Accès SSH (pour déboguer)
terminus connection:info SITE.live --fields=sftp_command
```

---

## Pantheon — QuickSilver Hooks

```php
// pantheon.yml — hooks automatiques (post-deploy, etc.)
```

```yaml
# pantheon.yml
api_version: 1
filemount: "files"

# Hooks exécutés automatiquement
workflows:
  deploy:
    after:
      - type: webphp
        description: 'Run Drush deploy after code push'
        script: private/scripts/quicksilver/drush_deploy.php

  sync_code:
    after:
      - type: webphp
        description: 'Rebuild caches'
        script: private/scripts/quicksilver/rebuild_cache.php
```

```php
// private/scripts/quicksilver/drush_deploy.php
<?php
echo shell_exec('drush deploy 2>&1');
echo shell_exec('drush pm:security --format=json 2>&1');
```

---

## Acquia Cloud — acli

```bash
# Installer Acquia CLI
curl -OL https://github.com/acquia/cli/releases/latest/download/acli.phar
chmod +x acli.phar && mv acli.phar /usr/local/bin/acli

# Authentification
acli auth:login

# Lister les applications
acli api:applications:list

# === Déploiement ===

# Déployer un tag sur l'environnement de production
acli api:environments:code-switch ENVIRONMENT_ID --vcs-path=tags/1.2.0

# Variables d'environnement
acli api:environments:variables:list ENVIRONMENT_ID
acli api:environments:variables:create ENVIRONMENT_ID \
  --name=DRUPAL_HASH_SALT \
  --value="valeur-secrete" \
  --is-sensitive=1

# Drush via acli
acli remote:drush ENVIRONMENT_ID -- deploy -y

# === Gestion DB ===

# Backup DB
acli api:environments:database-backups-create ENVIRONMENT_ID DATABASE_NAME

# Download backup
acli api:environments:database-backup-download ENVIRONMENT_ID DATABASE_NAME BACKUP_ID > prod.sql.gz

# Copier DB prod → staging
acli api:environments:database-copy SOURCE_ENV_ID DEST_ENV_ID DATABASE_NAME

# === Pipelines CI/CD (Acquia Pipelines) ===
# Fichier : acquia-pipelines.yaml à la racine du projet
```

```yaml
# acquia-pipelines.yaml
version: 1.0.0
services:
  mysql:
    version: '5.7'

events:
  push:
    branches:
      ignore:
        - develop
  tag:

steps:
  - step:
      name: "Build"
      entrypoint: "composer"
      arguments:
        - install
        - --no-dev
        - --optimize-autoloader

  - step:
      name: "Test"
      entrypoint: "vendor/bin/phpunit"
      arguments:
        - --testsuite=unit

  - step:
      name: "Deploy"
      entrypoint: "bash"
      arguments:
        - deploy.sh
```

---

## Drush Aliases — Commandes Multi-Environnements

```php
// drush/sites/mon-projet.site.yml

local:
  root: '/var/www/html/web'
  uri: 'http://mon-projet.ddev.site'

# Pantheon
prod.pantheon:
  host: 'appserver.live.SITE_ID.drush.in'
  port: '2222'
  user: 'live.SITE_ID'
  root: '/code/web'
  uri: 'https://mon-site.com'
  paths:
    drush-script: 'drush'
  options:
    strict: 0

# Acquia
prod.acquia:
  host: 'mon-site.ssh.enterprise-g1.acquia-sites.com'
  user: 'mon-site.prod'
  root: '/var/www/html/mon-site.prod/docroot'
  uri: 'https://mon-site.com'
```

```bash
# Utilisation des aliases
drush @prod.pantheon status
drush @prod.acquia deploy
drush sql:sync @prod.pantheon @local  # Copier DB prod → local
drush rsync @prod.acquia:%files @local:%files  # Copier fichiers
```
