---
name: drupal-composer — patches
description: Appliquer des patches Composer sur les modules Drupal avec cweagans/composer-patches - patches drupal.org, patches locaux, forks, et diagnostic des patches qui échouent.
---

# Patches Composer — Référence Complète

## Installation de cweagans/composer-patches

```bash
composer require cweagans/composer-patches

# Activer le plugin dans composer.json (si pas déjà fait)
"config": {
  "allow-plugins": {
    "cweagans/composer-patches": true
  }
}
```

---

## Appliquer un Patch depuis drupal.org

```json
// composer.json — section "extra"
{
  "extra": {
    "patches": {
      "drupal/paragraphs": {
        "Fix #1234567 - Paragraphs crash on save with nested translations":
          "https://www.drupal.org/files/issues/2024-01-15/paragraphs-fix-translation-3412345-12.patch"
      },
      "drupal/core": {
        "Fix #9876543 - Menu link access check fails for anonymous":
          "https://www.drupal.org/files/issues/2024-02-20/core-menu-link-9876543-8.patch"
      }
    }
  }
}
```

```bash
# Appliquer les patches
composer install
# OU si déjà installé
composer update drupal/paragraphs --no-install && composer install
```

**Convention de nommage :** `"Fix #ISSUE_ID - Description courte"` — inclure toujours le numéro d'issue pour traçabilité.

---

## Patch Local (fichier dans le projet)

```json
{
  "extra": {
    "patches": {
      "drupal/core": {
        "Custom fix for our specific use case":
          "patches/drupal-core-custom-fix.patch"
      }
    }
  }
}
```

```bash
# Créer un patch local depuis un diff
git diff vendor/drupal/MODULE/src/File.php > patches/mon-fix.patch

# OU créer un patch depuis deux fichiers
diff -u original.php modified.php > patches/mon-fix.patch

# Tester le patch manuellement
cd vendor/drupal/MODULE
patch -p1 < ../../../patches/mon-fix.patch --dry-run  # Tester
patch -p1 < ../../../patches/mon-fix.patch             # Appliquer
```

---

## Patches via `patches-file`

Pour les projets avec beaucoup de patches, externaliser dans un fichier séparé :

```json
// composer.json
{
  "extra": {
    "patches-file": "composer.patches.json"
  }
}
```

```json
// composer.patches.json
{
  "patches": {
    "drupal/paragraphs": {
      "Fix nested translation crash": "https://www.drupal.org/files/..."
    },
    "drupal/views": {
      "Fix AJAX filter with multiple selections": "patches/views-ajax-fix.patch"
    }
  }
}
```

---

## `composer-patches` v2 — Format Étendu (plus robuste)

La v2 est stable depuis octobre 2025 (`2.0.0`). Elle conserve la clé `extra.patches`
et reste rétrocompatible avec le format court de la v1 (`"description": "url"`).
Sa nouveauté est le **format étendu** : une **liste d'objets** par package, qui permet
`sha256` (intégrité), `depth` et `extra` par patch.

```bash
# Installer la v2 (stable)
composer require "cweagans/composer-patches:^2.0"
```

```json
{
  "extra": {
    "patches": {
      "drupal/paragraphs": [
        {
          "description": "Fix #3412345 - translation crash",
          "url": "https://www.drupal.org/files/issues/2024-01-15/paragraphs-fix-3412345-12.patch",
          "sha256": "abc123...",
          "depth": 1
        }
      ]
    }
  }
}
```

> ⚠️ Pièges de migration v1 → v2 :
> - La clé reste `extra.patches` (et non `extra.composer-patches`).
> - Le champ d'URL s'appelle `url` (et non `source`).
> - Le format étendu est une **liste** `[ { ... } ]`, pas un objet `{ "desc": { ... } }`.
> - v2 génère un `patches.lock.json` à committer pour des builds reproductibles.

---

## Forks — Alternative aux Patches Lourds

Pour des modifications importantes d'un module contrib, forker est mieux qu'un patch :

```json
// composer.json — utiliser un fork GitHub/GitLab
{
  "repositories": [
    {
      "type": "vcs",
      "url": "https://github.com/MON_ORG/paragraphs"
    }
  ],
  "require": {
    "drupal/paragraphs": "dev-fix-translation-bug as 1.15.0"
    // "dev-BRANCH_NAME as VERSION_ORIGIN"
  }
}
```

---

## Diagnostic — Patches qui Échouent

```bash
# Voir les détails des patches appliqués
composer install -v 2>&1 | grep -A2 "Applying patch\|Could not apply"

# Forcer la réapplication des patches
composer install --prefer-source  # Télécharge la source complète (pas le dist)

# Variable pour plus de verbosité
COMPOSER_PROCESS_TIMEOUT=600 composer install -vvv 2>&1 | grep -A5 "patch"

# Diagnostiquer un patch qui échoue
cd vendor/drupal/paragraphs
# Tenter d'appliquer manuellement avec options de fuzz
patch -p1 --fuzz=3 < ../../../patches/paragraphs-fix.patch --dry-run

# Le patch ne correspond plus à la version → chercher une version mise à jour
# → https://www.drupal.org/project/ISSUE_ID → télécharger la dernière révision
```

---

## Gérer les Patches après Mise à Jour

```bash
# Après mise à jour d'un module, vérifier que les patches s'appliquent toujours
composer update drupal/paragraphs 2>&1 | grep -E "patch|Error"

# Si un patch est inclus dans la nouvelle version → le supprimer de composer.json
composer show drupal/paragraphs  # Voir le CHANGELOG de la nouvelle version

# Automatiser la vérification en CI
composer install --no-interaction 2>&1 | grep -i "could not apply patch"
# → Sortie non-nulle si un patch échoue = CI échoue
```

---

## Commandes Utiles

```bash
# Lister tous les patches déclarés dans composer.json
composer config extra.patches 2>/dev/null || jq '.extra.patches' composer.json

# Via drush — voir les modules avec patches
drush php:eval "
\$extra = json_decode(file_get_contents('composer.json'), TRUE);
\$patches = \$extra['extra']['patches'] ?? [];
foreach (\$patches as \$package => \$patch_list) {
  echo \$package . ': ' . count(\$patch_list) . ' patch(es)' . PHP_EOL;
}
"
```
