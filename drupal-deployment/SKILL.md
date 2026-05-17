---
name: drupal-deployment
description: Use when deploying Drupal sites to production environments - setting up GitLab CI/CD pipelines with SSH deploy (the dominant pattern for French agencies on VPS/OVH/Hetzner), using drush deploy (updb + cim + cr in correct order), implementing zero-downtime atomic deployments with symlinks, configuring environment-specific settings.php with getenv(), configuring Platform.sh (platform.app.yaml, routes.yaml, services.yaml, CLI commands), deploying on Pantheon (Terminus CLI, Drush aliases, multidev), deploying on Acquia Cloud (acli, deploy hooks, pipelines), configuring maintenance mode during deployments, running post-deploy verification, implementing rollback procedures, or managing database syncing between environments in Drupal 8-11+
---

# Drupal Deployment — Référence Complète

## Overview

Référentiel complet du déploiement Drupal 8-11+ : CI/CD GitLab/GitHub Actions sur VPS auto-hébergé (pattern dominant en agence française), déploiement zéro-temps-d'arrêt avec symlinks, `drush deploy`, settings.php par environnement, procédures de rollback. Couvre aussi les plateformes managées Platform.sh, Pantheon et Acquia.

## 🎯 Choisir la Stratégie de Déploiement

```
Auto-hébergé VPS (OVH, Hetzner, Scaleway) — Pattern dominant agences FR
  → GitLab CI/CD + SSH deploy → script drush deploy (le plus courant)
  → Déploiement atomique avec symlinks (zéro-downtime sur gros sites)

Hébergement managé Drupal (moins courant en France)
  → Platform.sh : git push + pipeline automatique (le plus flexible)
  → Pantheon : Terminus + QuickSilver hooks
  → Acquia : acli + pipelines (le plus enterprise)

Kubernetes / Docker Swarm (projets avancés)
  → kubectl apply + drush deploy

La commande commune à TOUT déploiement Drupal :
  drush deploy  (= updb + cim + cr dans l'ordre correct)
```

---

## Quick Decision Table

| Besoin | Outil | Référence |
|--------|-------|-----------|
| **CI/CD GitLab sur VPS (pattern agence FR)** | `.gitlab-ci.yml` → SSH → `drush deploy` | [cicd-pipelines.md](cicd-pipelines.md) |
| **CI/CD GitHub Actions sur VPS** | `.github/workflows/deploy.yml` | [cicd-pipelines.md](cicd-pipelines.md) |
| **Variables CI/CD secrètes** | GitLab CI → Settings → Variables (SSH_KEY, DB_PASS...) | [cicd-pipelines.md](cicd-pipelines.md) |
| Déploiement zéro-downtime (VPS gros trafic) | Script atomic avec symlinks | [zero-downtime.md](zero-downtime.md) |
| Activer le maintenance mode | `drush state:set system.maintenance_mode 1` | [zero-downtime.md](zero-downtime.md) |
| Appliquer les updates DB + config + cache | `drush deploy` | [zero-downtime.md](zero-downtime.md) |
| Déployer en production sans interruption | Atomic symlink swap | [zero-downtime.md](zero-downtime.md) |
| Settings.php par environnement | `getenv('ENVIRONMENT')` + includes | [zero-downtime.md](zero-downtime.md) |
| Post-deploy vérification automatique | `drush core:requirements --severity=2` | [zero-downtime.md](zero-downtime.md) |
| Rollback rapide | `git checkout TAG + composer install + drush deploy` | [zero-downtime.md](zero-downtime.md) |
| Synchroniser DB entre environments | `drush sql:sync @prod @local` | [zero-downtime.md](zero-downtime.md) |
| Audit post-déploiement | `drush pm:security + drush core:requirements` | [zero-downtime.md](zero-downtime.md) |
| Déployer sur Platform.sh | `git push + .platform.app.yaml` | [platform-sh.md](platform-sh.md) |
| Variables d'env Platform.sh | `platform variable:set` / `PLATFORM_VARIABLES` | [platform-sh.md](platform-sh.md) |
| Déployer sur Pantheon | `terminus env:deploy` | [acquia-pantheon.md](acquia-pantheon.md) |
| Déployer sur Acquia | `acli push:artifact` | [acquia-pantheon.md](acquia-pantheon.md) |
| Déployer sur Kubernetes | `kubectl apply + drush deploy` | [acquia-pantheon.md](acquia-pantheon.md) |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| `drush cim` avant `drush updb` | `drush deploy` (ordre correct : updb → cim → cr) | Config importée avant que le schéma soit à jour |
| Modifier des fichiers en production via FTP | Git + deploy pipeline | Modifications perdues au prochain déploiement |
| `composer update` en production | `composer install` (respecte le lock) | Versions inattendues installées |
| Pas de backup avant `drush updb` | `drush sql:dump` avant chaque déploiement | Irrécupérable si update échoue |
| `drush deploy` sans maintenance mode sur gros sites | Maintenance mode pendant updb sur DB > 1Go | Erreurs pour les visiteurs pendant la migration |
| Secrets en clair dans composer.json ou .env commité | Variables d'environnement CI/CD ou vault | Fuite de credentials |
| Pas de post-deploy check | `drush core:requirements --severity=2` | Problèmes silencieux en production |

## Évolution par Version Majeure

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| `drush deploy` | ❌ | ✅ Drush 10+ | ✅ | ✅ |
| `hook_deploy_N` | ❌ | ✅ D9.3+ | ✅ | ✅ |
| Platform.sh support | ✅ | ✅ | ✅ | ✅ |
| Pantheon Terminus | ✅ | ✅ | ✅ | ✅ |
| Acquia CLI (acli) | ❌ | ✅ | ✅ | ✅ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Incidents de déploiement réels.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions.

## See Also

- `drupal-config` — drush cim, drush deploy, config management workflow
- `drupal-composer` — composer install en production, --no-dev, --optimize-autoloader
- `drupal-docker` — Docker CI/CD, multi-stage Dockerfile, production
- `drupal-security` — Secrets, permissions, trusted_host_patterns
- `drupal-migration` — drush updb, hook_deploy_N, rollback
