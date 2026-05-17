---
name: drupal-composer — version constraints
description: Contraintes de version Composer pour Drupal - syntaxe ^, ~, *, ranges, stabilité, résolution de conflits, et upgrade de Drupal core.
---

# Contraintes de Version Composer — Référence Complète

## Syntaxe des Contraintes

```
^10.3.0    = >=10.3.0 <11.0.0  (compatible semver — recommandé)
~10.3.0    = >=10.3.0 <10.4.0  (patch seulement)
10.3.*     = >=10.3.0 <10.4.0  (identique à ~10.3.0)
>=10.3.0   = minimum seulement (dangereux — pas de plafond)
10.3.0     = exactement cette version (trop restrictif)
*          = n'importe quelle version (très dangereux)
10.3.0 || ^11  = version exacte OU plage
```

**Pour les modules Drupal contrib :**
```
^1.0   → installe 1.x.y (pas 2.0)
^2.0   → installe 2.x.y (pas 3.0)
~1.5   → installe 1.5.x (patch seulement)
```

---

## Contraintes Recommandées par Cas

```json
{
  "require": {
    // Core — utiliser ^ avec version mineure spécifique
    "drupal/core-recommended": "^10.3",

    // Module contrib stable — ^ est standard
    "drupal/paragraphs": "^1.15",

    // Module avec breaking changes fréquents — ~ plus strict
    "drupal/commerce": "~4.0",

    // Module en développement (beta/alpha) — contrainte explicite
    "drupal/next": "^1.6",

    // Module custom en monorepo — version exacte
    "mon-org/mon-module-custom": "1.2.0",

    // Bibliothèque PHP — semver classique
    "firebase/php-jwt": "^6.0",

    // Drush — version compatible Drupal 10
    "drush/drush": "^12"
  }
}
```

---

## Versions de Stabilité

```
@stable    = versions stables uniquement (défaut)
@RC        = Release Candidates acceptées
@beta      = bêtas acceptées
@alpha     = alphas acceptées
@dev       = branches dev acceptées (⚠️ instable)
```

```json
// Forcer une version minimum avec stabilité
"drupal/next": "^1.6@beta",
"drupal/search_api_solr": "^4.3@RC",

// Configuration globale dans composer.json
"minimum-stability": "stable",
"prefer-stable": true
```

---

## Upgrade Drupal Core — Commandes Exactes

### D9 → D10

```bash
# 1. Vérifier la compatibilité des modules
composer require drupal/upgrade_status --dev
drush en upgrade_status -y
drush upgrade_status:analyze --all

# 2. Upgrade core + dépendances
composer require \
  "drupal/core-recommended:^10" \
  "drupal/core-composer-scaffold:^10" \
  "drupal/core-project-message:^10" \
  "drush/drush:^12" \
  --update-with-all-dependencies

# 3. Mettre à jour aussi les modules contrib compatibles D10
composer update --with-all-dependencies

# 4. Appliquer les updates DB
drush updb -y && drush cim -y && drush cr
```

### D10 → D11

```bash
composer require \
  "drupal/core-recommended:^11" \
  "drupal/core-composer-scaffold:^11" \
  "drupal/core-project-message:^11" \
  "drush/drush:^13" \
  --update-with-all-dependencies
```

---

## `composer why-not` — Diagnostiquer les Conflits

```bash
# Pourquoi Drupal 10.3 ne peut pas être installé ?
composer why-not drupal/core:^10.3

# Exemple de sortie :
# drupal/paragraphs  1.14.0  requires  drupal/core (>=8.5 <10)
# → Le module paragraphs est trop vieux pour D10

# Solution : mettre à jour le module bloquant
composer require drupal/paragraphs:^1.15 --no-update
composer update drupal/paragraphs drupal/core-recommended --with-all-dependencies

# Diagnostiquer plusieurs packages simultanément
composer why-not drupal/core:^11 2>&1 | head -30
```

---

## Résoudre les Conflits de Dépendances

```bash
# 1. Identifier le conflit
composer install 2>&1 | head -30
# → "Your requirements could not be resolved to an installable set of packages."

# 2. Diagnostiquer avec why-not
composer why-not drupal/core:^10

# 3. Options de résolution

# OPTION A : mettre à jour les packages bloquants
composer require drupal/MODULE_BLOQUANT:^NEW_VERSION

# OPTION B : contraindre plus strictement
composer require drupal/core-recommended:"^10.2.6 !=10.2.7"

# OPTION C : utiliser --ignore-platform-reqs (⚠️ dev uniquement)
composer install --ignore-platform-reqs

# OPTION D : voir le graphe complet des dépendances
composer depends drupal/core --tree
```

---

## `composer.lock` — Gestion et Bonnes Pratiques

```bash
# composer.lock = snapshot exact de toutes les versions installées
# TOUJOURS committer composer.lock en production → reproductibilité

# Regénérer composer.lock depuis composer.json
composer update --lock    # ← Only updates the lock file hash

# Vérifier que lock et json sont cohérents
composer validate --strict

# Installer EXACTEMENT les versions du lock (CI, production)
composer install  # → utilise le lock, pas le json
```

**Bonnes pratiques :**
- ✅ Committer `composer.lock` toujours
- ✅ `composer install` en CI/prod (respecte le lock)
- ❌ `composer update` en prod (ignore le lock)
- ❌ `composer.lock` dans `.gitignore`
