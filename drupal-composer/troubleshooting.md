---
name: drupal-composer — troubleshooting
description: Diagnostiquer et résoudre les problèmes Composer sur les projets Drupal - conflits de dépendances, mémoire insuffisante, patches qui échouent, et timeout.
---

# Composer Troubleshooting — Référence Complète

## Erreurs les Plus Fréquentes

### "Your requirements could not be resolved"

```bash
# Diagnostic — voir exactement ce qui bloque
composer why-not drupal/core:^10 2>&1 | head -30

# La sortie indique : PACKAGE VERSION requires AUTRE_PACKAGE (CONTRAINTE_INCOMPATIBLE)
# Exemple :
# drupal/paragraphs 1.14.0 requires drupal/core (>=8.5 <10)
# → paragraphs 1.14 n'est pas compatible D10

# Solution A : mettre à jour le package bloquant
composer require drupal/paragraphs:^1.15 --no-update
composer update drupal/paragraphs --with-all-dependencies

# Solution B : forcer la mise à jour avec toutes les dépendances
composer update --with-all-dependencies

# Solution C : voir tous les conflits à la fois
composer update --dry-run 2>&1 | grep "- Conflict"
```

### Mémoire Insuffisante

```bash
# Erreur : "Allowed memory size of X bytes exhausted"

# Solution 1 : augmenter la mémoire pour cette commande
COMPOSER_MEMORY_LIMIT=-1 composer update

# Solution 2 : via php.ini temporairement
php -d memory_limit=-1 /usr/local/bin/composer update

# Solution 3 : dans composer.json
{
  "config": {
    "process-timeout": 600
  }
}

# Solution 4 : dans ~/.composer/config.json (global)
{
  "config": {
    "process-timeout": 600
  }
}
```

### Timeout sur les Downloads

```bash
# Erreur : "The process ... exceeded the timeout of 300 seconds"

# Augmenter le timeout
COMPOSER_PROCESS_TIMEOUT=600 composer install

# Ou configurer globalement
composer config --global process-timeout 600

# Pour les packages lents (téléchargement très lent)
composer install --prefer-dist  # archives zip (plus rapide que git clone)
```

### Patch qui Échoue

```bash
# Erreur : "Could not apply patch"

# Voir plus de détails
COMPOSER_PROCESS_TIMEOUT=600 composer install -v 2>&1 | grep -A5 "Could not apply"

# Tester le patch manuellement
cd vendor/drupal/paragraphs
patch -p1 --dry-run < ../../../patches/mon-fix.patch

# Options si le patch est "fuzzable"
patch -p1 --fuzz=3 < ../../../patches/mon-fix.patch --dry-run

# Vérifier si le fix est inclus dans la nouvelle version
composer show drupal/paragraphs  # voir le CHANGELOG de la version installée

# Mettre à jour le patch : chercher la version mise à jour sur drupal.org
# → https://www.drupal.org/project/paragraphs/issues/ISSUE_ID
```

---

## Diagnostics Généraux

```bash
# Valider composer.json et composer.lock
composer validate --strict

# Vérifier que vendor/ est cohérent avec composer.lock
composer check-platform-reqs

# Voir la version de Composer
composer --version

# Mettre à jour Composer lui-même
composer self-update

# Vider le cache Composer (résout des problèmes obscurs)
composer clear-cache

# Afficher le graphe des dépendances
composer depends drupal/core
composer depends drupal/paragraphs --tree

# Voir les packages installés
composer show --installed | grep drupal/

# Voir uniquement les packages directs (pas les dépendances transitives)
composer show --direct
```

---

## vendor/ Corrompu ou Incohérent

```bash
# Supprimer vendor/ et réinstaller proprement
rm -rf vendor/
composer install

# ⚠️ Ne jamais faire en production — risque d'interruption de service
# En production : déployer depuis un build artifact CI

# Vérifier l'intégrité
composer check-platform-reqs
php -r "require 'vendor/autoload.php'; echo 'Autoloader OK';"
```

---

## Conflits après Mise à Jour de Drupal Core

```bash
# Après bump de version majeure — voir tous les incompatibles
composer update drupal/core drupal/core-recommended \
  drupal/core-composer-scaffold drupal/core-project-message \
  --with-all-dependencies --dry-run 2>&1 | grep -E "CONFLICT|downgrade"

# Trouver les modules incompatibles avec la nouvelle version
composer why-not drupal/core:^11 2>&1

# Mettre à jour un module incompatible spécifique
composer require drupal/MODULE:^NEW_VERSION --no-update
composer update drupal/MODULE drupal/core-recommended \
  --with-all-dependencies

# Vérifier les modules sans version D11 compatible
composer outdated --direct 2>&1 | grep drupal/
```

---

## Packages Manquants ou "Not Found"

```bash
# Erreur : "Package drupal/MODULE not found"

# Vérifier le repository
composer config repositories

# Ajouter le repository Drupal si absent
composer config repositories.drupal \
  composer https://packages.drupal.org/8

# Vérifier que le package existe vraiment
curl https://packages.drupal.org/8/p2/drupal/MODULE.json | jq '.packages | keys'

# Si c'est un package privé — vérifier l'authentification
cat ~/.composer/auth.json  # credentials configurés ?
echo $COMPOSER_AUTH        # variable d'environnement ?
```

---

## Composer dans Docker — Problèmes Fréquents

```bash
# Composer dans un container sans cache → très lent
# Monter le cache global
docker run --rm \
  -v $(pwd):/app \
  -v ~/.composer:/root/.composer \
  composer:2 install

# Ou dans docker-compose.yml :
volumes:
  - ~/.composer:/home/www-data/.composer

# Permissions dans le container
docker compose exec --user www-data php composer install

# composer.lock non respecté → forcer
docker compose exec php composer install --no-interaction
# (pas composer update !)
```

---

## Platform Requirements

```bash
# Voir les requirements de plateforme requis
composer check-platform-reqs

# Ignorer les requirements (dev uniquement — jamais prod)
composer install --ignore-platform-reqs

# Simuler une plateforme différente (ex: prod sur PHP 8.3)
composer install --ignore-platform-req=php
COMPOSER_IGNORE_PLATFORM_REQS=1 composer install

# Configurer les requirements attendus dans composer.json
{
  "config": {
    "platform": {
      "php": "8.3.0"
    }
  }
}
```
