# Gestion des Assets — libraries.yml

## Structure d'une Librairie

```yaml
# mon_theme.libraries.yml

global-styling:
  version: 1.0
  css:
    base:           # Niveau 1 — variables, reset, typographie fondamentale
      css/base/variables.css: {}
      css/base/reset.css: {}
    layout:         # Niveau 2 — grille, régions, conteneurs
      css/layout/grid.css: {}
      css/layout/regions.css: {}
    component:      # Niveau 3 — boutons, cartes, formulaires
      css/components/button.css: {}
      css/components/card.css: {}
    state:          # Niveau 4 — états (hover, focus, disabled, active)
      css/states/focus.css: {}
    theme:          # Niveau 5 — couleurs, polices — surcharges finales
      css/theme/colors.css: {}
      css/theme/typography.css: {}
  dependencies:
    - core/normalize   # Reset CSS de base

global-scripts:
  version: 1.0
  js:
    js/navigation.js: {}
    js/utils.js: { weight: -5 }    # Charger avant les autres JS du thème
  dependencies:
    - core/drupal          # Objet Drupal global + translations
    - core/once            # Remplace jQuery.once() — D9+

# Librairie conditionnelle (chargée depuis Twig ou preprocess)
modal:
  version: 1.0
  css:
    component:
      css/components/modal.css: {}
  js:
    js/modal.js: {}
  dependencies:
    - core/drupal
    - core/drupalSettings  # Accès à drupalSettings côté JS

# Librairie externe (CDN)
fontawesome:
  version: 6.4
  css:
    theme:
      https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css:
        { type: external, minified: true }

# CSS compilé depuis SCSS/PostCSS (workflow de build séparé)
compiled:
  version: 1.0
  css:
    theme:
      dist/css/main.css: { minified: true }
  js:
    dist/js/bundle.js: { minified: true }
```

---

## Options des Assets

```yaml
css/components/modal.css:
  {}                       # Défaut — préprocessé, agrégé

css/vendor/legacy.css:
  { preprocess: false }    # Exclure de l'agrégation/minification Drupal (fichiers déjà minifiés ou avec @import custom)

dist/css/styles.min.css:
  { minified: true }       # Déclaré comme déjà minifié

js/polyfill.js:
  { weight: -100 }         # Charger très tôt (poids bas = avant)

js/deferred.js:
  { weight: 100 }          # Charger tard

js/tracking.js:
  { attributes: { defer: true, async: true } }  # Attributs HTML sur la balise <script>

js/critical.js:
  { header: true }         # Charger dans <head> au lieu du footer (défaut)

https://external.cdn/lib.js:
  { type: external, minified: true }

css/print.css:
  { media: print }         # CSS chargé uniquement pour l'impression
```

---

## Dépendances Core — Quoi utiliser

| Dépendance | Fournit | Quand |
|-----------|---------|-------|
| `core/drupal` | `Drupal` global, `Drupal.t()`, `Drupal.behaviors` | Toujours si JS Drupal |
| `core/once` | `once()` — attacher un behavior une seule fois | Remplace `jQuery.once()` — D9+ |
| `core/drupalSettings` | `drupalSettings` objet PHP→JS | Passer données PHP vers JS |
| `core/jquery` | jQuery | **Éviter en général** — non chargé par défaut D9+. Exception : si base theme Bootstrap5 ou contrib jQuery-dépendant |
| `core/drupal.once` | ❌ **Alias invalide** | Utiliser `core/once` — `core/drupal.once` n'est pas le bon identifiant |
| `core/normalize` | normalize.css | Reset CSS cross-browser |
| `core/drupal.dialog` | Dialogues Drupal (UI jQuery) | Modales Drupal natives |

---

## 3 Façons d'Attacher une Librairie

### 1. Globale — `.info.yml`

```yaml
libraries:
  - mon_theme/global-styling    # Chargée sur TOUTES les pages
  - mon_theme/global-scripts
```

### 2. Conditionnelle depuis Twig

```twig
{# Charger uniquement sur ce template #}
{{ attach_library('mon_theme/modal') }}

{# Conditionnel en Twig #}
{% if node.bundle == 'article' %}
  {{ attach_library('mon_theme/article-styles') }}
{% endif %}
```

### 3. Depuis PHP (preprocess ou module)

```php
// Dans un preprocess du .theme
function mon_theme_preprocess_node(&$variables) {
  if ($variables['node']->bundle() === 'article') {
    $variables['#attached']['library'][] = 'mon_theme/article-styles';
  }
}

// Dans un render array de module
$build['#attached']['library'][] = 'mon_theme/modal';
```

---

## `libraries-override` et `libraries-extend`

```yaml
# Dans mon_theme.info.yml
# ⚠️ YAML : une seule clé libraries-override par fichier — tout combiné sous la même clé

libraries-override:
  # Remplacer entièrement une librairie
  core/drupal.dialog:
    mon_theme/custom-dialog: {}
  # Désactiver un fichier CSS spécifique
  core/normalize:
    css:
      base:
        assets/vendor/normalize-css/normalize.css: false
  # Désactiver une librairie entière
  system/base: false

# ÉTENDRE une librairie existante (ajouter sans remplacer)
libraries-extend:
  core/drupal.dialog:
    - mon_theme/dialog-extend    # Nos styles s'ajoutent au dialog Drupal
  user/drupal.user:
    - mon_theme/user-styles
```

---

## `drupalSettings` — Passer des Données PHP → JS

```php
// Dans preprocess ou module
function mon_theme_preprocess_page(&$variables) {
  $variables['#attached']['drupalSettings']['monTheme'] = [
    'apiUrl'  => '/api/v1',
    'userId'  => \Drupal::currentUser()->id(),
    'lang'    => \Drupal::languageManager()->getCurrentLanguage()->getId(),
  ];
  $variables['#attached']['library'][] = 'mon_theme/global-scripts';
}
```

```javascript
// Dans js/navigation.js — dépendance core/drupalSettings requise
(function (Drupal, drupalSettings, once) {
  Drupal.behaviors.monThemeNav = {
    attach: function (context, settings) {
      // settings === drupalSettings
      const apiUrl = settings.monTheme.apiUrl;

      once('mon-theme-nav', '[data-nav]', context).forEach(function (el) {
        // S'exécute UNE SEULE fois par élément (équivalent jQuery.once)
        el.addEventListener('click', function () {
          fetch(apiUrl + '/items').then(/* ... */);
        });
      });
    }
  };
})(Drupal, drupalSettings, once);
```

---

## Pattern Drupal Behaviors (JS moderne)

```javascript
// js/mon-comportement.js
// Dépendances : core/drupal + core/once

(function (Drupal, once) {

  'use strict';

  Drupal.behaviors.monComportement = {

    attach: function (context, settings) {
      // once() garantit l'exécution unique même après AJAX
      once('mon-comportement', '.mon-selecteur', context).forEach(function (element) {
        element.addEventListener('click', function (e) {
          e.preventDefault();
          // Logique...
        });
      });
    },

    detach: function (context, settings, trigger) {
      // Optionnel — nettoyage avant suppression du contexte (AJAX, etc.)
      if (trigger === 'unload') {
        // Nettoyer les event listeners si nécessaire
      }
    }
  };

})(Drupal, once);
```

**Pourquoi `Drupal.behaviors` ?**
- S'exécute au chargement de page ET après chaque requête AJAX
- `context` = l'élément DOM nouvellement ajouté (pas forcément `document`)
- `once()` empêche les double-attachements

---

## Webpack Mix (Laravel Mix) — Outil Legacy Courant

Encore très utilisé dans les projets Drupal existants (`webpack.mix.js`) :

```bash
# package.json
{
  "scripts": {
    "dev": "mix",
    "watch": "mix watch",
    "build": "NODE_ENV=production mix"
  },
  "devDependencies": {
    "laravel-mix": "^6.0",
    "sass": "^1.69.0"
  }
}
```

```javascript
// webpack.mix.js
const mix = require('laravel-mix');

mix
  .js('src/scripts/app.js', 'dist/app.js')
  .sass('src/styles/app.scss', 'dist/app.css')
  .options({ processCssUrls: false });
```

Le CSS/JS compilé va dans `dist/` → référencé dans `.libraries.yml`.

---

## Workflow Build Tools (SCSS/Vite/Webpack)

```json
// package.json (exemple simplifié)
{
  "scripts": {
    "build": "vite build",
    "watch": "vite build --watch",
    "dev": "vite"
  },
  "devDependencies": {
    "vite": "^5.0.0",
    "sass": "^1.69.0"
  }
}
```

Le CSS/JS compilé va dans `dist/` → référencé dans `.libraries.yml` avec `minified: true`.

```bash
# Développement
npm run watch

# Production
npm run build
docker compose exec php drush cr   # Pour que Drupal prenne en compte les nouveaux fichiers
```
