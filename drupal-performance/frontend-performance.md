---
name: drupal-performance — frontend performance
description: Optimisation des performances frontend pour Drupal - agrégation CSS/JS, Core Web Vitals (LCP, CLS, INP), lazy loading images, preload, critical CSS, et font-display.
---

# Frontend Performance Drupal — Référence Complète

## Agrégation CSS/JS

### Activer en Production

```bash
# Via drush
drush config:set system.performance css.preprocess 1 -y
drush config:set system.performance js.preprocess 1 -y

# OU via settings.php (surcharge la config)
$config['system.performance']['css']['preprocess'] = TRUE;
$config['system.performance']['js']['preprocess'] = TRUE;

# Après modification → vider le cache
drush cr
```

**Effet :** Drupal combine et minifie tous les CSS/JS en fichiers uniques avec hash de cache-busting.

### Désactiver en Développement

```php
// settings.local.php
$config['system.performance']['css']['preprocess'] = FALSE;
$config['system.performance']['js']['preprocess'] = FALSE;

// ← permet de voir les erreurs JS/CSS individuellement
```

### Exclure un Fichier de l'Agrégation

```yaml
# mon_theme.libraries.yml — marquer un fichier pour ne pas être agrégé
global:
  css:
    theme:
      css/critique.css:
        preprocess: false   # Ne pas inclure dans l'agrégation
  js:
    dist/dynamic.js:
      preprocess: false
```

---

## Core Web Vitals — Optimisation Drupal

### LCP (Largest Contentful Paint) — Cible : < 2.5s

```php
// 1. Preload de l'image hero (LCP element)
function mon_theme_preprocess_page(array &$variables): void {
  // Identifier l'image hero de la page courante
  if ($node = \Drupal::routeMatch()->getParameter('node')) {
    if ($node->hasField('field_image_hero') && !$node->get('field_image_hero')->isEmpty()) {
      $media = $node->get('field_image_hero')->entity;
      if ($media) {
        $file = $media->get('field_media_image')->entity;
        $uri = $file->getFileUri();
        $url = \Drupal::service('file_url_generator')->generateAbsoluteString($uri);

        // Ajouter le preload dans le head
        $variables['#attached']['html_head'][] = [
          [
            '#tag' => 'link',
            '#attributes' => [
              'rel' => 'preload',
              'as' => 'image',
              'href' => $url,
              'fetchpriority' => 'high',
            ],
          ],
          'hero-preload',
        ];
      }
    }
  }
}
```

```twig
{# Dans le template — ajouter fetchpriority="high" sur l'image LCP #}
{% if is_front or is_hero_page %}
  <img
    src="{{ hero_image_url }}"
    alt="{{ hero_image_alt }}"
    fetchpriority="high"
    loading="eager"
    width="1200"
    height="600"
  >
{% else %}
  <img
    src="{{ image_url }}"
    alt="{{ image_alt }}"
    loading="lazy"
    width="{{ image_width }}"
    height="{{ image_height }}"
  >
{% endif %}
```

### CLS (Cumulative Layout Shift) — Cible : < 0.1

```twig
{# TOUJOURS spécifier width et height sur les images → évite le CLS #}
{# ❌ CLS garanti #}
<img src="{{ url }}" alt="...">

{# ✅ Dimensions réservées #}
<img src="{{ url }}" alt="..." width="800" height="450" loading="lazy">

{# Dans le responsive images template — aspect-ratio CSS #}
<div style="aspect-ratio: 16/9; overflow: hidden;">
  {{ content.field_image }}
</div>
```

```css
/* Prévenir le CLS pour les fonts */
@font-face {
  font-family: 'MonFont';
  src: url('/fonts/monfont.woff2') format('woff2');
  font-display: swap;  /* Affiche le fallback immédiatement, swap quand chargée */
  font-weight: 400;
  font-style: normal;
}
```

### INP (Interaction to Next Paint) — Cible : < 200ms

```javascript
// Defer le JavaScript non-critique
// Dans mon_theme.libraries.yml :
// js:
//   dist/non-critical.js:
//     attributes:
//       defer: true
//     preprocess: false

// Pattern Drupal behaviors correct — initialiser dans requestAnimationFrame
(function (Drupal, once) {
  'use strict';

  Drupal.behaviors.monAnimationLourde = {
    attach(context, settings) {
      once('mon-animation', '[data-animation]', context).forEach(function (el) {
        // Différer les animations non-critiques
        requestAnimationFrame(() => {
          el.classList.add('animation-ready');
        });
      });
    },
  };
})(Drupal, once);
```

---

## Lazy Loading des Images

```php
// Drupal D10+ — lazy loading natif sur les images
// hook_preprocess_image_formatter
function mon_module_preprocess_image_formatter(array &$variables): void {
  $item = $variables['item'];

  // Ajouter loading="lazy" par défaut
  if (!isset($variables['image']['#attributes']['loading'])) {
    $variables['image']['#attributes']['loading'] = 'lazy';
  }

  // Ajouter les dimensions pour éviter le CLS
  if ($item->width && $item->height) {
    $variables['image']['#attributes']['width'] = $item->width;
    $variables['image']['#attributes']['height'] = $item->height;
  }
}
```

```yaml
# Via la configuration du formatter image dans le display mode
# /admin/structure/types/manage/article/display
# Image → Formatter settings → Enable lazy loading
```

---

## Preconnect & DNS Prefetch

```php
// Précharger les domaines tiers (fonts.googleapis.com, CDN, analytics...)
function mon_theme_preprocess_html(array &$variables): void {
  // DNS prefetch — résolution DNS anticipée
  $variables['#attached']['html_head'][] = [
    [
      '#tag' => 'link',
      '#attributes' => ['rel' => 'dns-prefetch', 'href' => '//fonts.googleapis.com'],
    ],
    'dns-prefetch-google-fonts',
  ];

  // Preconnect — établir la connexion TCP/TLS à l'avance
  $variables['#attached']['html_head'][] = [
    [
      '#tag' => 'link',
      '#attributes' => [
        'rel' => 'preconnect',
        'href' => 'https://fonts.gstatic.com',
        'crossorigin' => '',
      ],
    ],
    'preconnect-gstatic',
  ];

  // Preload d'une police
  $variables['#attached']['html_head'][] = [
    [
      '#tag' => 'link',
      '#attributes' => [
        'rel' => 'preload',
        'as' => 'font',
        'href' => '/themes/custom/mon_theme/fonts/monfont.woff2',
        'type' => 'font/woff2',
        'crossorigin' => '',
      ],
    ],
    'preload-font-monfont',
  ];
}
```

---

## WebP — Images Modernes

```bash
# Drupal D10+ — WebP natif dans les Image Styles
# /admin/config/media/image-styles → créer un style → ajouter "Convert" → WebP

# PHP — vérifier le support WebP
drush php:eval "echo function_exists('imagewebp') ? 'WebP: OK' : 'WebP: MANQUE libgd';"

# Dockerfile — s'assurer que GD supporte WebP
RUN docker-php-ext-configure gd --with-jpeg --with-webp --with-freetype
RUN docker-php-ext-install gd
```

```yaml
# Image style avec conversion WebP automatique
# config/install/image.style.article_thumbnail.yml
effects:
  uuid-scale:
    id: image_scale_and_crop
    weight: 0
    data:
      width: 400
      height: 300
  uuid-convert:
    id: image_convert
    weight: 1
    data:
      extension: webp
      quality: 85
```

---

## Critical CSS — Above the Fold

```php
// Inliner le CSS critique directement dans <head> (zero render-blocking)
function mon_theme_preprocess_html(array &$variables): void {
  $critical_css_path = \Drupal::root() . '/themes/custom/mon_theme/css/critical.css';

  if (file_exists($critical_css_path)) {
    $variables['#attached']['html_head'][] = [
      [
        '#tag' => 'style',
        '#attributes' => ['id' => 'critical-css'],
        '#value' => file_get_contents($critical_css_path),
      ],
      'critical-css',
    ];
  }

  // Charger le CSS non-critique de manière asynchrone
  $variables['#attached']['html_head'][] = [
    [
      '#tag' => 'link',
      '#attributes' => [
        'rel' => 'preload',
        'as' => 'style',
        'href' => '/themes/custom/mon_theme/dist/css/main.css',
        'onload' => "this.onload=null;this.rel='stylesheet'",
      ],
    ],
    'async-main-css',
  ];
}
```

---

## AdvAgg — Optimisation Avancée CSS/JS

Le module `drupal/advagg` (Advanced CSS/JS Aggregation) améliore l'agrégation native Drupal.

```bash
composer require drupal/advagg
drush en advagg -y
# Configuration : /admin/config/development/performance/advagg
```

**Ce qu'advagg apporte vs l'agrégation Drupal native :**

| Feature | Drupal natif | advagg |
|---------|-------------|--------|
| Agrégation | ✅ | ✅ + fichiers plus petits |
| Minification JS | ✅ basique | ✅ JSMin/Terser |
| Minification CSS | ✅ basique | ✅ CSSMin |
| Subresource Integrity | ❌ | ✅ (SHA hash sur les assets) |
| Cache busting intelligent | ❌ | ✅ hash du contenu |
| HTTP/2 Server Push hints | ❌ | ✅ optionnel |
| CDN rewriting | ❌ | ✅ optionnel |

```php
// AdvAgg avec CDN — réécrire les URLs CSS/JS vers un CDN
// /admin/config/development/performance/advagg/cdn
// CDN URL : https://cdn.mon-site.com

// AdvAgg Subresource Integrity — sécurité renforcée
// Chaque asset CSS/JS reçoit un hash SHA-384 en attribut integrity
// → Protection contre les attaques CDN supply chain
```

---

## Audit Performance Drupal — Checklist

```bash
# 1. Agrégation CSS/JS active
drush config:get system.performance | grep preprocess

# 2. Page Cache actif
drush pm:list --status=enabled | grep page_cache

# 3. BigPipe actif
drush pm:list --status=enabled | grep big_pipe

# 4. Images avec dimensions
# → Vérifier manuellement dans Lighthouse

# 5. Lighthouse score
# → Google PageSpeed Insights ou Lighthouse CLI
npx lighthouse https://mon-site.com --output html --output-path report.html

# 6. Core Web Vitals via Chrome DevTools
# → Performance tab → Web Vitals : LCP, INP, CLS
```
