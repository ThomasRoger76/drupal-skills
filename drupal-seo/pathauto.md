---
name: drupal-seo — pathauto et redirects
description: Configurer drupal/pathauto pour les alias URL Drupal (patterns avec tokens), drupal/redirect pour les 301/302, et gérer les redirections en masse.
---

# Pathauto & Redirects — Référence Complète

## Installation

```bash
composer require drupal/pathauto drupal/redirect drupal/token
drush en pathauto redirect token -y
```

---

## Pathauto — Patterns d'Alias URL

```
/admin/config/search/path/patterns/add

Configuration d'un pattern :
  ├── Type d'entité : Content (node)
  ├── Bundle : Article
  ├── Pattern : articles/[node:title]
  │             → /articles/mon-titre-darticle
  └── Alias existant : Aucune action | Créer nouvel alias | Supprimer l'existant
```

### Patterns Recommandés

```
# Nœuds par type de contenu
articles/[node:title]               → /articles/mon-article
produits/[node:field_categorie:entity:title]/[node:title]
                                    → /produits/electromenager/mon-produit
[node:field_rubrique:entity:url:path]/[node:title]
                                    → URL relative à la rubrique parente

# Termes de taxonomie
[term:vocabulary]/[term:name]       → /categories/drupal
[term:parents:join-path]/[term:name] → /dev/php/drupal (hiérarchique)

# Utilisateurs
utilisateurs/[user:name]            → /utilisateurs/jean-dupont
```

### Options Importantes

```
/admin/config/search/path/settings

✅ Automatic alias : générer alias automatiquement à la sauvegarde
✅ Update action : "Create a new alias. Delete the old alias."
✅ Transliterate prior to creating alias : convertir les accents
   → "é" → "e", "ç" → "c"
✅ Reduce strings to letters and numbers : supprimer la ponctuation
✅ Separator : - (tiret)
✅ Maximum alias length : 100
```

---

## Tokens Pathauto Custom

```php
<?php
/**
 * Implements hook_token_info().
 */
function mon_module_token_info(): array {
  $info = [];

  $info['tokens']['node']['seo-slug'] = [
    'name' => t('SEO Slug'),
    'description' => t('Slug SEO optimisé (sans stop words).'),
  ];

  return $info;
}

/**
 * Implements hook_tokens().
 */
function mon_module_tokens(string $type, array $tokens, array $data, array $options): array {
  $replacements = [];

  if ($type === 'node' && isset($data['node'])) {
    $node = $data['node'];

    foreach ($tokens as $name => $original) {
      if ($name === 'seo-slug') {
        $title = $node->getTitle();
        // Supprimer les mots vides français
        $stop_words = ['le', 'la', 'les', 'de', 'du', 'des', 'un', 'une', 'et', 'ou'];
        $words = explode(' ', strtolower($title));
        $words = array_filter($words, fn($w) => !in_array($w, $stop_words) && strlen($w) > 2);
        $replacements[$original] = implode('-', array_slice($words, 0, 5));
      }
    }
  }

  return $replacements;
}
```

---

## Commandes Drush Pathauto

```bash
# Générer/régénérer les alias pour toutes les entités
drush pathauto:aliases-generate

# Générer pour un type spécifique
drush pathauto:aliases-generate node article

# Générer pour les entités sans alias existant
drush pathauto:aliases-generate --no-existing

# Vérifier l'état des aliases
drush php:eval "
\$alias_manager = \Drupal::service('path_alias.manager');
echo \$alias_manager->getAliasByPath('/node/42') . PHP_EOL;
"

# Régénérer après import de contenu (migration)
drush pathauto:aliases-generate node article && echo "Aliases générés"
```

---

## Redirect — Gestion des 301/302

```bash
# Ajouter une redirection manuelle
# /admin/config/search/redirect/add

# Ou via Drush
drush php:eval "
\$redirect = \Drupal\redirect\Entity\Redirect::create([
  'redirect_source' => ['path' => 'ancien-chemin', 'query' => []],
  'redirect_redirect' => ['uri' => 'internal:/node/42'],
  'status_code' => 301,
  'language' => 'fr',
]);
\$redirect->save();
echo 'Redirection créée : ' . \$redirect->id();
"
```

### Import de Redirections en Masse

```bash
# Module redirect_import
composer require drupal/redirect_import
drush en redirect_import -y

# Format CSV : source_path,destination_path,status_code
# /old-page,/new-page,301
# /old-category/old-article,/new-category/new-article,301

# Import via UI : /admin/config/search/redirect/import
# OU via Drush :
drush redirect-import /path/to/redirects.csv --format=csv
```

---

## Intégration Pathauto + Redirect

Quand un alias Pathauto change, le module Redirect crée automatiquement une redirection 301 de l'ancien vers le nouvel alias :

```
Configuration : /admin/config/search/path/settings
→ "When an alias is changed, redirect the old alias to the new one" : ✅
```

**Exemple :**
```
Article "Mon Article" créé → /articles/mon-article
Article renommé "Mon Super Article" → /articles/mon-super-article
Redirect automatique : /articles/mon-article → 301 → /articles/mon-super-article
```

---

## Troubleshooting

```bash
# Alias non généré → vérifier le pattern
drush php:eval "
\$patterns = \Drupal::entityTypeManager()->getStorage('pathauto_pattern')->loadMultiple();
foreach (\$patterns as \$p) {
  echo \$p->id() . ': ' . \$p->getPattern() . ' [' . \$p->getType() . ']' . PHP_EOL;
}
"

# Alias cassé (404) → vérifier dans la table path_alias
drush sql-query "SELECT * FROM path_alias WHERE path = '/node/42'"

# Vider le cache des aliases
drush php:eval "\Drupal::service('path_alias.manager')->cacheClear();"
drush cr
```

---

## Robots.txt — Gestion dans Drupal

### Protéger le robots.txt du scaffold

```json
// composer.json — empêcher le scaffold d'écraser robots.txt
"extra": {
  "drupal-scaffold": {
    "file-mapping": {
      "[web-root]/robots.txt": false
    }
  }
}
```

### robots.txt Drupal recommandé

```
# web/robots.txt — version standard Drupal

User-agent: *

# Ne pas indexer les pages d'administration
Disallow: /admin/
Disallow: /user/
Disallow: /node/add/
Disallow: /*/edit
Disallow: /*/delete

# Ne pas indexer les URLs de recherche et filtrées
Disallow: /search/
Disallow: /*?*

# Ne pas indexer les fichiers système
Disallow: /core/
Disallow: /themes/
Disallow: /modules/
Disallow: /profiles/
Disallow: /vendor/

# Indexer les assets publics
Allow: /themes/*/assets/
Allow: /themes/*/images/
Allow: /themes/*/dist/

# Sitemap
Sitemap: https://mon-site.com/sitemap.xml
```

### Module `drupal/robotstxt` (via UI)

```bash
composer require drupal/robotstxt
drush en robotstxt -y
# Configurer via /admin/config/search/robotstxt
# Permet de modifier robots.txt depuis l'interface sans toucher aux fichiers
```

### robots.txt par Environnement

```php
// settings.php — désactiver l'indexation en staging/dev
// Via Metatag global settings
$config['metatag.metatag_defaults.global']['tags']['robots'] = 'noindex, nofollow';

// OU via le fichier robots.txt staging
// → Bloquer tous les robots en staging
// User-agent: *
// Disallow: /
```
