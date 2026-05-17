---
name: drupal-deployment — CI/CD pipelines
description: Pipelines CI/CD automatisés pour Drupal - GitLab CI et GitHub Actions avec build, test, et déploiement automatique.
---

# CI/CD Pipelines — GitLab CI & GitHub Actions

## GitLab CI — Pipeline Complet Drupal

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - test
  - build
  - deploy

variables:
  COMPOSER_CACHE_DIR: "$CI_PROJECT_DIR/.composer-cache"

cache:
  key: composer-$CI_COMMIT_REF_SLUG
  paths:
    - .composer-cache/

# ── VALIDATE ──────────────────────────────────────────────────────────────

validate:security:
  stage: validate
  script:
    - composer audit --no-dev
    - vendor/bin/drush pm:security --format=json
  allow_failure: false

validate:phpcs:
  stage: validate
  script:
    - vendor/bin/phpcs --standard=Drupal web/modules/custom web/themes/custom

# ── TEST ──────────────────────────────────────────────────────────────────

test:unit:
  stage: test
  script:
    - composer install --no-interaction
    - vendor/bin/phpunit --testsuite=unit
  artifacts:
    reports:
      junit: phpunit-unit.xml

# ── BUILD ─────────────────────────────────────────────────────────────────

build:production:
  stage: build
  script:
    - composer install --no-dev --optimize-autoloader
    - npm run build   # Si build frontend
  artifacts:
    paths:
      - vendor/
      - web/themes/custom/*/dist/
    expire_in: 1 hour
  only:
    - main
    - tags

# ── DEPLOY ────────────────────────────────────────────────────────────────

deploy:staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.mon-site.com
  script:
    - ssh deploy@staging.mon-site.com "cd /var/www/staging && git pull && composer install --no-dev && drush deploy -y"
  only:
    - develop

deploy:production:
  stage: deploy
  environment:
    name: production
    url: https://mon-site.com
  script:
    - ssh deploy@mon-site.com "bash /var/www/deploy.sh"
  only:
    - main
    - tags
  when: manual   # Déploiement production = validation manuelle
```

---

## GitHub Actions — Déploiement Automatique

```yaml
# .github/workflows/deploy.yml
name: Deploy Drupal

on:
  push:
    branches: [main]
  release:
    types: [published]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Cache Composer
        uses: actions/cache@v4
        with:
          path: ~/.composer/cache
          key: composer-${{ hashFiles('**/composer.lock') }}

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'

      - name: Install dependencies
        run: composer install --no-interaction --prefer-dist

      - name: Security check
        run: composer audit --no-dev

  deploy-production:
    needs: validate
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://mon-site.com
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan mon-site.com >> ~/.ssh/known_hosts

      - name: Deploy
        run: |
          ssh deploy@mon-site.com "
            cd /var/www/mon-site &&
            git pull origin main &&
            composer install --no-dev --optimize-autoloader --no-interaction &&
            vendor/bin/drush deploy -y &&
            vendor/bin/drush core:requirements --severity=2
          "

      - name: Health check
        run: |
          HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" https://mon-site.com/)
          [ "$HTTP_CODE" = "200" ] || exit 1
```

---

## Variables CI/CD à Configurer

| Variable | Description | Sensible |
|----------|-------------|---------|
| `SSH_PRIVATE_KEY` | Clé SSH pour accès au serveur | ✅ |
| `MAILER_DSN` | Transport email production | ✅ |
| `DB_PASSWORD` | Mot de passe base de données | ✅ |
| `DRUPAL_HASH_SALT` | Hash salt Drupal | ✅ |
| `COMPOSER_AUTH` | Credentials repos privés | ✅ |
| `DEPLOY_HOST` | Hostname du serveur | — |

```bash
# GitLab CI — ajouter une variable
# Settings → CI/CD → Variables → Add Variable
# OU via l'API GitLab

# GitHub Actions
# Settings → Secrets and variables → Actions → New repository secret
```
