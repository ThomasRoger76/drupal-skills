---
name: drupal-composer — deployment
description: Composer dans les workflows de déploiement Drupal - CI/CD, cache, production optimizations, scripts post-deploy, et Docker.
---

# Composer et Déploiement — Référence Complète

## Stratégie de Déploiement avec Composer

```
Développement local       CI/CD Pipeline           Production
composer install          composer install          composer install
  (avec dev deps)         --no-dev                  --no-dev
                          --optimize-autoloader     --optimize-autoloader
                          → tests → build           → drush deploy
```

---

## Commande de Déploiement Production

```bash
# La commande complète pour la production
composer install \
  --no-dev \                    # Pas de dépendances de développement
  --optimize-autoloader \       # Classmap pré-calculé
  --no-interaction \            # Aucune question interactive
  --no-progress \               # Pas de barre de progression (CI)
  --prefer-dist                 # Archives zip plutôt que git clone

# Après Composer → déploiement Drupal
vendor/bin/drush deploy  # updb + cim + cr dans l'ordre correct
```

---

## Cache Composer en CI/CD

```yaml
# .gitlab-ci.yml — cache pour accélérer les builds
cache:
  key: composer-$CI_COMMIT_REF_SLUG
  paths:
    - .composer-cache/
  policy: pull-push

variables:
  COMPOSER_CACHE_DIR: .composer-cache  # Stocké dans le workspace GitLab

before_script:
  - composer install --no-interaction --prefer-dist
```

```yaml
# GitHub Actions — cache Composer
- name: Get Composer cache directory
  id: composer-cache
  run: echo "dir=$(composer config cache-files-dir)" >> $GITHUB_OUTPUT

- uses: actions/cache@v4
  with:
    path: ${{ steps.composer-cache.outputs.dir }}
    key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
    restore-keys: ${{ runner.os }}-composer-

- name: Install dependencies
  run: composer install --prefer-dist --no-progress --no-interaction
```

---

## Cache Composer dans Docker

```yaml
# docker-compose.yml — monter le cache Composer global
services:
  php:
    volumes:
      - .:/var/www/html
      - ${PERSONAL_GLOBAL_COMPOSER_FOLDER:-~/.composer}:/home/www-data/.composer
      # → Évite de re-télécharger les packages à chaque `composer install`
```

```dockerfile
# Dockerfile — optimiser le layer cache Composer
FROM php:8.3-apache AS base

COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer

WORKDIR /var/www/html

# ← Copier UNIQUEMENT les fichiers Composer en premier (meilleur cache Docker)
COPY composer.json composer.lock ./

# Cette couche est cachée si composer.json/lock n'ont pas changé
RUN composer install --no-dev --optimize-autoloader --no-interaction --no-progress

# Puis copier le reste du code
COPY . .
```

---

## Scripts Composer pour l'Automatisation

```json
// composer.json
{
  "scripts": {
    "post-install-cmd": [
      "DrupalProject\\composer\\ScriptHandler::createRequiredFiles"
    ],
    "post-update-cmd": [
      "DrupalProject\\composer\\ScriptHandler::createRequiredFiles"
    ],

    // Scripts custom pour le workflow
    "site-install": [
      "drush site:install standard -y",
      "@php -r \"file_put_contents('web/sites/default/settings.local.php', '');\"",
      "drush cr"
    ],
    "site-update": [
      "drush updb -y",
      "drush cim -y",
      "drush cr"
    ],
    "verify": [
      "drush core:requirements --severity=2",
      "drush pm:security"
    ],
    "fix-permissions": [
      "chmod -R 755 web/sites/default/files",
      "chown -R www-data:www-data web/sites/default/files"
    ]
  }
}
```

```bash
# Exécuter les scripts
composer site-install
composer site-update
composer verify
```

---

## Déploiement Zéro-Temps d'Arrêt (Atomic)

```bash
#!/bin/bash
# deploy.sh — déploiement atomique

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RELEASE_DIR="/var/www/releases/$TIMESTAMP"
CURRENT_LINK="/var/www/current"
SHARED_DIR="/var/www/shared"

# 1. Créer le nouveau release directory
mkdir -p "$RELEASE_DIR"

# 2. Cloner ou copier le code
git clone --depth=1 origin main "$RELEASE_DIR"

# 3. Liens symboliques vers les fichiers partagés (non versionnés)
ln -s "$SHARED_DIR/files" "$RELEASE_DIR/web/sites/default/files"
ln -s "$SHARED_DIR/settings.local.php" "$RELEASE_DIR/web/sites/default/settings.local.php"

# 4. Installer les dépendances
cd "$RELEASE_DIR"
composer install --no-dev --optimize-autoloader --no-interaction

# 5. Maintenance mode avant la migration
$CURRENT_LINK/vendor/bin/drush state:set system.maintenance_mode 1 -y

# 6. Basculer le lien symbolique (atomique — moins d'1ms de downtime)
ln -sfn "$RELEASE_DIR" "$CURRENT_LINK"

# 7. Appliquer les mises à jour Drupal
"$CURRENT_LINK/vendor/bin/drush" deploy

# 8. Désactiver le maintenance mode
"$CURRENT_LINK/vendor/bin/drush" state:set system.maintenance_mode 0 -y

# 9. Garder les N derniers releases
ls -t /var/www/releases | tail -n +6 | xargs -I {} rm -rf "/var/www/releases/{}"

echo "Déploiement $TIMESTAMP terminé."
```

---

## Vérifications Post-Déploiement

```bash
# Vérifier l'état du site après déploiement
drush core:requirements --severity=2  # Problèmes critiques uniquement

# Vérifier les modules vulnérables
drush pm:security

# Vérifier la configuration est bien importée
drush config:status

# Vérifier les caches fonctionnent
curl -I https://mon-site.com/ | grep X-Drupal-Cache

# Vérifier que le site répond
curl -s -o /dev/null -w "%{http_code}" https://mon-site.com/
```
