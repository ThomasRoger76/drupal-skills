---
name: drupal-deployment — zero downtime
description: Déploiement zéro-temps-d'arrêt pour Drupal - atomic symlinks, drush deploy, maintenance mode, settings.php par environnement, et rollback.
---

# Déploiement Zéro-Temps-d'Arrêt — Référence Complète

## La Commande Universelle : `drush deploy`

```bash
# drush deploy = séquence correcte en une seule commande
drush deploy

# Équivaut à (ordre garanti par Drush) :
drush updatedb -y          # 1. hook_update_N (schéma) — AVANT la config
drush config:import -y     # 2. Importer la config YAML (sur un schéma à jour)
drush cache:rebuild        # 3. Reconstruire les caches
drush deploy:hook -y       # 4. hook_deploy_NAME (logique post-config : data migration…)

# ⚠️ L'ordre est crucial — ne jamais faire cim avant updb
# 💡 D11 / Drush 12+ : les hooks de déploiement sont nommés
#    function HOOK_deploy_set_default_value(&$sandbox) dans MODULE.deploy.php
#    (le suffixe numéroté hook_deploy_N reste supporté).
#    hook_deploy_NAME tourne APRÈS cim — idéal pour manipuler des données
#    qui dépendent de la config fraîchement importée.
```

---

## Settings.php par Environnement

```php
// web/sites/default/settings.php — pattern multi-environnement

// Config de base (commune à tous les environnements)
$settings['config_sync_directory'] = '../config/sync';
$settings['hash_salt'] = getenv('DRUPAL_HASH_SALT') ?: 'VALEUR_LOCALE';

// Détecter l'environnement
$environment = getenv('APP_ENV') ?: 'local';

// Inclure les settings de l'environnement si disponibles
$env_settings = __DIR__ . "/settings.$environment.php";
if (file_exists($env_settings)) {
  include $env_settings;
}

// Settings local (jamais commité)
if (file_exists(__DIR__ . '/settings.local.php')) {
  include __DIR__ . '/settings.local.php';
}
```

```php
// web/sites/default/settings.production.php
$databases['default']['default'] = [
  'driver' => 'mysql',
  'database' => getenv('DB_NAME'),
  'username' => getenv('DB_USER'),
  'password' => getenv('DB_PASSWORD'),
  'host' => getenv('DB_HOST') ?: 'localhost',
  'port' => getenv('DB_PORT') ?: '3306',
  'prefix' => '',
];

$settings['trusted_host_patterns'] = [
  '^mon-site\.com$',
  '^www\.mon-site\.com$',
];

// Redis en production
$settings['cache']['default'] = 'cache.backend.redis';
$settings['redis.connection']['host'] = getenv('REDIS_HOST') ?: 'redis';

// Pas d'erreurs visibles en production
$config['system.logging']['error_level'] = 'hide';
```

---

## Script de Déploiement Atomique (VPS)

```bash
#!/bin/bash
# deploy.sh — déploiement zéro-temps-d'arrêt avec symlinks atomiques

set -e  # Arrêter sur la première erreur

DEPLOY_DIR="/var/www/releases"
SHARED_DIR="/var/www/shared"
CURRENT_LINK="/var/www/current"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RELEASE="$DEPLOY_DIR/$TIMESTAMP"

# Filet de sécurité : désactiver le maintenance mode même si le script échoue
# (cf. lessons.md — maintenance mode oublié actif après déploiement)
trap '[ -L "$CURRENT_LINK" ] && "$CURRENT_LINK/vendor/bin/drush" \
  state:set system.maintenance_mode 0 --input-format=integer -y || true' ERR

echo "=== Déploiement $TIMESTAMP ==="

# 0. Backup DB AVANT toute modification (cf. lessons.md — rollback impossible)
if [ -L "$CURRENT_LINK" ]; then
  "$CURRENT_LINK/vendor/bin/drush" sql:dump --gzip \
    --result-file="$SHARED_DIR/backups/db_$TIMESTAMP.sql"
fi

# 1. Créer le répertoire de release
mkdir -p "$RELEASE"

# 2. Cloner depuis git (ou copier depuis artifact CI)
git clone --depth=1 --branch main https://github.com/mon-org/mon-site.git "$RELEASE"
# OU : rsync depuis CI artifact

# 3. Liens symboliques vers les fichiers partagés
ln -s "$SHARED_DIR/files" "$RELEASE/web/sites/default/files"
ln -s "$SHARED_DIR/settings.local.php" "$RELEASE/web/sites/default/settings.local.php"
ln -s "$SHARED_DIR/.env" "$RELEASE/.env"

# 4. Installer les dépendances
cd "$RELEASE"
composer install --no-dev --optimize-autoloader --no-interaction

# 5. Maintenance mode AVANT le switch (si gros site avec updb long)
if [ -L "$CURRENT_LINK" ]; then
  "$CURRENT_LINK/vendor/bin/drush" state:set system.maintenance_mode 1 --input-format=integer -y
fi

# 6. Switch atomique du symlink (presque zéro downtime)
ln -sfn "$RELEASE" "$CURRENT_LINK"

# 7. Déployer (updb + cim + cr)
"$CURRENT_LINK/vendor/bin/drush" deploy -y

# 8. Désactiver maintenance mode
"$CURRENT_LINK/vendor/bin/drush" state:set system.maintenance_mode 0 --input-format=integer -y

# 9. Vérifications post-déploiement
"$CURRENT_LINK/vendor/bin/drush" core:requirements --severity=2
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" https://mon-site.com/)
echo "HTTP: $HTTP_CODE"
[ "$HTTP_CODE" = "200" ] || { echo "ERREUR: Site retourne $HTTP_CODE"; exit 1; }

# 10. Garder seulement les 5 dernières releases
ls -t "$DEPLOY_DIR" | tail -n +6 | while read old; do
  rm -rf "$DEPLOY_DIR/$old"
  echo "Release supprimée: $old"
done

echo "=== Déploiement $TIMESTAMP terminé ==="
```

---

## Rollback

```bash
#!/bin/bash
# rollback.sh — revenir à la release précédente

DEPLOY_DIR="/var/www/releases"
CURRENT_LINK="/var/www/current"

# Trouver la release précédente
CURRENT=$(readlink "$CURRENT_LINK")
PREVIOUS=$(ls -t "$DEPLOY_DIR" | sed -n '2p')

if [ -z "$PREVIOUS" ]; then
  echo "ERREUR: Pas de release précédente disponible"
  exit 1
fi

echo "Rollback: $CURRENT → $DEPLOY_DIR/$PREVIOUS"

# Switch atomique
ln -sfn "$DEPLOY_DIR/$PREVIOUS" "$CURRENT_LINK"

# Re-appliquer la config et vider le cache
"$CURRENT_LINK/vendor/bin/drush" deploy -y

echo "Rollback terminé vers $PREVIOUS"
```

---

## Maintenance Mode

```bash
# Activer le mode maintenance
drush state:set system.maintenance_mode 1 --input-format=integer -y
# OU
drush php:eval "\Drupal::state()->set('system.maintenance_mode', TRUE);"

# Désactiver
drush state:set system.maintenance_mode 0 --input-format=integer -y

# Vérifier
drush php:eval "echo \Drupal::state()->get('system.maintenance_mode') ? 'ON' : 'OFF';"

# Message custom pendant la maintenance
drush config:set system.maintenance '{"message":"Mise à jour en cours. Retour dans 5 minutes."}' -y
```

---

## Sync DB Entre Environnements

```bash
# Copier la DB de production vers local (avec Drush aliases)
drush sql:sync @prod @local

# Copier les fichiers
drush rsync @prod:%files @local:%files

# Sans alias — via dump/restore
ssh prod "drush sql:dump --gzip | base64" | base64 -d | gunzip -c | drush sql:cli

# Avec Docker natif (jamais DDEV)
docker compose exec php drush sql:sync @prod @self
```
