---
name: drupal-composer — basics
description: Structure du composer.json Drupal, commandes essentielles, scaffold, plugins Composer Drupal, et gestion du vendor/.
---

# Composer Drupal — Fondamentaux

> **Currency D11 :** le template ci-dessous est en `^10` (encore le plus déployé).
> Pour un nouveau projet Drupal 11, remplacer chaque `^10` par `^11`, `drush/drush`
> par `^13`, et viser PHP 8.3+ (`config.platform.php`). Voir la procédure exacte
> D10→D11 dans [version-constraints.md](version-constraints.md).

## Structure `composer.json` Drupal Standard

```json
{
  "name": "mon-organisation/mon-projet",
  "description": "Site Drupal de Mon Organisation",
  "type": "project",
  "license": "GPL-2.0-or-later",
  "require": {
    "composer/installers": "^2.0",
    "cweagans/composer-patches": "^1.7",
    "drupal/core-composer-scaffold": "^10",
    "drupal/core-project-message": "^10",
    "drupal/core-recommended": "^10.3",
    "drush/drush": "^12"
  },
  "require-dev": {
    "drupal/core-dev": "^10",
    "phpunit/phpunit": "^10",
    "mglaman/phpstan-drupal": "^1.2",
    "palantirnet/drupal-rector": "^0.20"
  },
  "config": {
    "allow-plugins": {
      "composer/installers": true,
      "cweagans/composer-patches": true,
      "drupal/core-composer-scaffold": true,
      "drupal/core-project-message": true,
      "phpstan/extension-installer": true,
      "dealerdirect/phpcodesniffer-composer-installer": true
    },
    "sort-packages": true,
    "optimize-autoloader": true,
    "preferred-install": "dist"
  },
  "extra": {
    "drupal-scaffold": {
      "locations": {
        "web-root": "web/"
      },
      "file-mapping": {
        "[web-root]/.htaccess": false,          ← Protéger le .htaccess custom
        "[web-root]/robots.txt": false,          ← Protéger le robots.txt custom
        "[web-root]/sites/default/settings.php": false
      }
    },
    "installer-paths": {
      "web/core": ["type:drupal-core"],
      "web/libraries/{$name}": ["type:drupal-library"],
      "web/modules/contrib/{$name}": ["type:drupal-module"],
      "web/profiles/contrib/{$name}": ["type:drupal-profile"],
      "web/themes/contrib/{$name}": ["type:drupal-theme"],
      "drush/Commands/contrib/{$name}": ["type:drupal-drush"]
    },
    "patches": {}
  },
  "scripts": {
    "post-install-cmd": [
      "@php ./vendor/bin/drush --yes deploy"
    ],
    "post-update-cmd": [],
    "drupal-scaffold": [
      "DrupalComposer\\DrupalScaffold\\Plugin::scaffold"
    ]
  }
}
```

---

## Commandes Essentielles

```bash
# ── Installation ────────────────────────────────────────────────────────────

# Premier install (crée vendor/)
composer install

# CI/Production — pas de dev, autoloader optimisé
composer install --no-dev --optimize-autoloader --no-interaction

# Avec le cache Composer (évite de re-télécharger)
COMPOSER_CACHE_DIR=~/.composer composer install

# ── Ajout de modules ────────────────────────────────────────────────────────

# Installer un module (dernière version stable)
composer require drupal/paragraphs

# Installer une version spécifique
composer require drupal/paragraphs:^1.15

# Installer en dev (pour les outils de développement)
composer require --dev drupal/devel

# Installer sans mettre à jour les autres packages
composer require drupal/metatag --no-update
composer update drupal/metatag  # Puis mettre à jour séparément

# ── Mise à jour ─────────────────────────────────────────────────────────────

# Mettre à jour un module spécifique
composer update drupal/paragraphs

# Mettre à jour Drupal core uniquement
composer update drupal/core "drupal/core-*" --with-all-dependencies

# Voir ce qui peut être mis à jour
composer outdated --direct

# ── Suppression ─────────────────────────────────────────────────────────────

composer remove drupal/MODULE_NAME

# ── Informations ────────────────────────────────────────────────────────────

# Voir les dépendances d'un package
composer show drupal/paragraphs

# Voir toutes les dépendances installées
composer show --installed | grep drupal/

# Voir pourquoi un package est installé
composer why drupal/token

# Voir pourquoi une version ne peut pas être installée
composer why-not drupal/paragraphs:^2.0
```

---

## `drupal/core-recommended` vs `drupal/core`

```json
// ✅ RECOMMANDÉ pour les projets
"drupal/core-recommended": "^10"

// Ce package inclut :
// → drupal/core (le vrai core)
// → toutes les dépendances core avec les versions testées par l'équipe Drupal
// → Évite les conflits de dépendances

// ⚠️ ALTERNATIF (projets avancés)
"drupal/core": "^10"
// → Seulement le core, dépendances libres de contraintes
// → Peut installer des versions incompatibles de Symfony, etc.
```

---

## Scaffold — Fichiers Auto-générés

`drupal/core-composer-scaffold` génère des fichiers dans le webroot au `composer install` :

```bash
# Fichiers générés par le scaffold :
web/.htaccess
web/index.php
web/robots.txt
web/sites/default/default.settings.php
web/sites/default/default.services.yml
# ... et d'autres

# Protéger un fichier custom contre l'écrasement :
"extra": {
  "drupal-scaffold": {
    "file-mapping": {
      "[web-root]/.htaccess": false,  ← false = ne pas générer ce fichier
      "[web-root]/robots.txt": {
        "path": "assets/robots.txt",  ← utiliser notre fichier custom
        "overwrite": false            ← ne jamais écraser
      }
    }
  }
}

# Après modification du scaffold config :
composer drupal:scaffold  # Ou : composer install (re-scaffold auto)
```

---

## Scripts Composer pour l'Automatisation

```json
"scripts": {
  "drupal-install": [
    "@php ./vendor/bin/drush site:install standard --account-name=admin --account-pass=admin -y",
    "@php ./vendor/bin/drush config:set system.site uuid $(grep uuid config/sync/system.site.yml | awk '{print $2}') -y",
    "@php ./vendor/bin/drush config:import -y",
    "@php ./vendor/bin/drush cache:rebuild"
  ],
  "drupal-update": [
    "@php ./vendor/bin/drush updatedb -y",
    "@php ./vendor/bin/drush config:import -y",
    "@php ./vendor/bin/drush cache:rebuild"
  ],
  "lint": [
    "@php ./vendor/bin/phpcs --standard=Drupal web/modules/custom"
  ],
  "test": [
    "@php ./vendor/bin/phpunit --testsuite=unit"
  ]
}
```

```bash
# Exécuter un script
composer run drupal-install
composer run lint
composer run test
```

---

## Autoloader — Optimisation Production

```bash
# Générer un autoloader optimisé (classmap + autorité de classe)
composer install --optimize-autoloader

# OU séparément
composer dump-autoload --optimize --no-dev

# Différence de performance :
# Développement : PSR-4 autoload (cherche les fichiers à la demande)
# Production avec --optimize : classmap pré-calculé → 10-30% plus rapide
```

---

## Vérifier l'Intégrité

```bash
# Vérifier que composer.lock est cohérent avec composer.json
composer validate

# Vérifier que vendor/ correspond à composer.lock
composer check-platform-reqs

# Vérifier les vulnérabilités connues
composer audit

# Vérifier les packages obsolètes
composer outdated
```
