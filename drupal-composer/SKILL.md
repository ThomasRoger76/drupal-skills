---
name: drupal-composer
description: Use when managing Drupal dependencies with Composer - applying patches with cweagans/composer-patches, configuring version constraints for Drupal core and contrib modules (^10, ~10.2, 10.2.*), setting up private repositories with authentication, managing composer.json for Drupal projects (drupal/core-recommended vs drupal/core), running composer scripts for deployment automation, resolving dependency conflicts, using composer why-not to diagnose version incompatibilities, configuring Composer plugins (drupal/core-composer-scaffold, dealerdirect/phpcodesniffer-composer-installer), handling Composer in Docker environments, or optimizing autoloader for production in Drupal 8-11+
---

# Drupal Composer — Référence Complète

## Overview

Référentiel complet de la gestion des dépendances Drupal avec Composer 2 : structure du `composer.json` Drupal, contraintes de version, patches, repositories privés, scripts, déploiement, et résolution de conflits.

## 🎯 La Règle Fondamentale

> **Composer est la seule source de vérité pour les dépendances.** Jamais de modules téléchargés manuellement, jamais de modifications directes dans `vendor/`. Tout passe par `composer.json` → commit → `composer install` en production.

---

## Quick Decision Table

| Besoin | Commande / Outil | Référence |
|--------|-----------------|-----------|
| Installer un module Drupal contrib | `composer require drupal/MODULE_NAME` | [composer-basics.md](composer-basics.md) |
| Installer en version spécifique | `composer require drupal/MODULE_NAME:^2.0` | [version-constraints.md](version-constraints.md) |
| Upgrade Drupal core | `composer require drupal/core-recommended:^10` | [version-constraints.md](version-constraints.md) |
| Appliquer un patch Drupal | `cweagans/composer-patches` + extra.patches | [patches.md](patches.md) |
| Patch depuis drupal.org | URL de l'issue + `forks:` dans composer.json | [patches.md](patches.md) |
| Forcer l'application d'un patch | `--prefer-source` ou `COMPOSER_MIRROR_PATH_REPOS=1` | [patches.md](patches.md) |
| Repository privé GitLab | `repositories` + `COMPOSER_AUTH` | [private-repos.md](private-repos.md) |
| Repository Composer privé (Satis/Packagist.com) | `type: composer` dans repositories | [private-repos.md](private-repos.md) |
| Modules custom dans un package Composer | `type: composer` sur le repo git du module | [private-repos.md](private-repos.md) |
| Voir pourquoi un package ne peut pas être installé | `composer why-not drupal/MODULE:^2` | [troubleshooting.md](troubleshooting.md) |
| Voir pourquoi un package est installé | `composer why drupal/MODULE` | [troubleshooting.md](troubleshooting.md) |
| Résoudre un conflit de dépendances | `composer update --with-all-dependencies` | [troubleshooting.md](troubleshooting.md) |
| Optimiser l'autoloader pour la production | `composer install --no-dev --optimize-autoloader` | [deployment.md](deployment.md) |
| Script de déploiement post-install | `scripts.post-install-cmd` dans composer.json | [deployment.md](deployment.md) |
| Vérifier les vulnérabilités PHP | `composer audit` | [security.md](security.md) |
| Garder Composer lui-même à jour | `composer self-update` | [composer-basics.md](composer-basics.md) |
| Scaffold Drupal (settings.php, .htaccess) | `drupal/core-composer-scaffold` plugin | [composer-basics.md](composer-basics.md) |
| Exclure un fichier du scaffold | `extra.drupal-scaffold.file-mapping` → false | [composer-basics.md](composer-basics.md) |
| Installer en lecture seule (CI) | `composer install --no-interaction` | [deployment.md](deployment.md) |
| Cache Composer dans Docker | Volume `~/.composer` monté | [deployment.md](deployment.md) |
| Diagnostiquer lenteur Composer | `COMPOSER_PROCESS_TIMEOUT=600` | [troubleshooting.md](troubleshooting.md) |
| Créer un plugin Composer custom | `type: composer-plugin` dans composer.json | [composer-basics.md](composer-basics.md) |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| Modifier des fichiers dans `vendor/` | Patches via `cweagans/composer-patches` | vendor/ est écrasé au prochain install |
| `composer update` sans contraintes | `composer update drupal/MODULE_NAME` spécifique | Mise à jour de toutes les dépendances = risque |
| `composer require drupal/core:dev-main` | `drupal/core-recommended:^10.3` | dev-main = instable |
| Committer `vendor/` dans git | `.gitignore` avec `vendor/` | Dépôt gonflé, conflits |
| `rm -rf vendor && composer install` en production | `composer install` (sans suppression) | Lent, pas atomique |
| Patch via `composer.json` sans numéro d'issue | URL complète drupal.org + description | Impossible à maintenir |
| `"drupal/MODULE": "*"` comme contrainte | `"drupal/MODULE": "^2.0"` | Installe n'importe quelle version |
| Credentials dans `composer.json` | `COMPOSER_AUTH` env var ou `auth.json` (gitignored) | Secrets exposés dans git |

## Évolution Composer × Drupal

| Feature | Composer 1 | Composer 2 | Drupal requis |
|---------|-----------|-----------|--------------|
| Vitesse | Lente | ✅ 2-5× plus rapide | D9+ (Composer 2 requis) |
| `drupal/core-recommended` | contrib | ✅ core recommandé | D8.8+ |
| `drupal/core-composer-scaffold` | contrib | ✅ | D8.8+ |
| Patches via composer-patches | ✅ | ✅ | Toutes versions |
| `composer audit` | ❌ | ✅ | Toutes versions |
| Parallel downloads | ❌ | ✅ | — |
| `--dry-run` amélioré | basique | ✅ | — |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Problèmes Composer résolus en projet réel.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions.

## See Also

- `drupal-docker` — Cache Composer dans Docker, Composer dans les containers
- `drupal-migration` — Composer lors des upgrades de version majeure
- `drupal-security` — `composer audit`, vulnérabilités PHP
- `drupal-deployment` — déploiement production, `composer install --no-dev` en CI/CD
- `drupal-testing` — Composer pour les dépendances de test (PHPUnit, PHPStan)
