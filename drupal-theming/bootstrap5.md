# Bootstrap 5 + Drupal — Guide Complet

## Installation du module contrib

```bash
composer require drupal/bootstrap5
docker compose exec php drush en bootstrap5 -y
docker compose exec php drush cr
```

Le module `drupal/bootstrap5` installe le thème contrib du même nom. Ce thème **ne doit jamais être modifié directement** — il faut impérativement créer un sous-thème.

---

## Créer un Sous-Thème Bootstrap 5

### Structure minimale

```
web/themes/custom/montheme/
├── montheme.info.yml          # Obligatoire — déclare le base theme: bootstrap5
├── montheme.libraries.yml     # Assets CSS/JS du sous-thème
├── montheme.theme             # Preprocess + hooks PHP
├── montheme.breakpoints.yml   # Optionnel — points de rupture responsives
├── config/
│   └── install/
│       └── montheme.settings.yml   # Paramètres Bootstrap5 du thème
├── css/
│   └── style.css              # CSS custom compilé ou écrit directement
├── js/
│   └── scripts.js             # JS custom
├── templates/                 # Templates Twig surchargés
│   ├── layout/
│   ├── content/
│   ├── block/
│   └── navigation/
└── images/
```

### `montheme.info.yml`

```yaml
name: Mon Thème Bootstrap 5
description: 'Sous-thème Bootstrap 5 custom.'
type: theme
core_version_requirement: ^10 || ^11
package: Custom

# OBLIGATOIRE — hériter de bootstrap5
base theme: bootstrap5

# Régions Bootstrap5 standard (adapter selon le projet)
regions:
  page_top: 'Page top'
  header: 'En-tête'
  primary_menu: 'Menu principal'
  secondary_menu: 'Menu secondaire'
  hero: 'Hero banner'
  highlighted: 'Mis en avant'
  help: 'Aide'
  breadcrumb: 'Fil d''Ariane'
  content: 'Contenu'
  sidebar_first: 'Barre latérale gauche'
  sidebar_second: 'Barre latérale droite'
  footer_first: 'Pied de page 1'
  footer_second: 'Pied de page 2'
  footer_third: 'Pied de page 3'
  page_bottom: 'Page bottom'

# Librairies globales du sous-thème
libraries:
  - montheme/global-styling
  - montheme/global-scripts

# Désactiver le CSS Bootstrap5 contrib si on gère Bootstrap depuis notre build
libraries-override:
  bootstrap5/bootstrap5.css: false
```

### `montheme.libraries.yml`

```yaml
global-styling:
  version: 1.0
  css:
    theme:
      css/style.css: {}
  dependencies:
    - bootstrap5/bootstrap5    # CSS+JS Bootstrap5 fourni par le thème contrib

global-scripts:
  version: 1.0
  js:
    js/scripts.js: {}
  dependencies:
    - core/drupal
    - core/once

# Librairie avec Bootstrap 5 compilé depuis le sous-thème (si workflow SCSS)
compiled:
  version: 1.0
  css:
    theme:
      dist/css/main.css: { minified: true }
  js:
    dist/js/bundle.js: { minified: true }
  dependencies:
    - core/drupal
    - core/once
```

### `montheme.theme`

```php
<?php

/**
 * @file
 * Fonctions de preprocess et hooks du thème Bootstrap 5 custom.
 */

use Drupal\Core\Form\FormStateInterface;

/**
 * Implements hook_preprocess_HOOK() for page.html.twig.
 */
function montheme_preprocess_page(&$variables) {
  // Ajouter des variables globales disponibles dans page.html.twig
  $variables['site_name'] = \Drupal::config('system.site')->get('name');
  $variables['is_front'] = \Drupal::service('path.matcher')->isFrontPage();
}

/**
 * Implements hook_preprocess_HOOK() for node.html.twig.
 */
function montheme_preprocess_node(&$variables) {
  $node = $variables['node'];
  // Ajouter une classe Bootstrap selon le type de contenu
  $variables['attributes']['class'][] = 'node--' . $node->bundle();
}

/**
 * Implements hook_preprocess_HOOK() for block.html.twig.
 */
function montheme_preprocess_block(&$variables) {
  // Ajouter l'ID du bloc comme attribut data pour le debug
  if (isset($variables['elements']['#id'])) {
    $variables['attributes']['data-block-id'] = $variables['elements']['#id'];
  }
}

/**
 * Implements hook_form_alter().
 * Ajouter les classes Bootstrap 5 aux formulaires Drupal.
 */
function montheme_form_alter(&$form, FormStateInterface $form_state, $form_id) {
  // Ajouter la classe Bootstrap au formulaire
  $form['#attributes']['class'][] = 'needs-validation';
}
```

### `config/install/montheme.settings.yml`

```yaml
# Paramètres Bootstrap5 du sous-thème
# Voir les options dans le module bootstrap5/config/schema/bootstrap5.schema.yml
fluid_container: false          # Container fluid ou fixe
navbar_collapse_breakpoint: lg  # Breakpoint du menu responsive (sm/md/lg/xl)
navbar_position: static         # static/fixed-top/fixed-bottom/sticky-top
scroll_spy: false               # ScrollSpy Bootstrap
mobile_menu_style: offcanvas    # offcanvas/collapse
breadcrumb_display: true        # Afficher le fil d'Ariane
```

---

## Configurer les Régions Bootstrap5

Le thème contrib `bootstrap5` fournit des templates `page.html.twig` qui utilisent les classes de grille Bootstrap. Pour personnaliser la mise en page :

```twig
{# templates/layout/page.html.twig — copié depuis bootstrap5 et adapté #}
<div class="container-{{ container_type|default('md') }}">
  <header class="site-header py-3">
    <div class="row">
      <div class="col-md-4">
        {{ page.header }}
      </div>
      <div class="col-md-8">
        {{ page.primary_menu }}
      </div>
    </div>
  </header>

  {% if page.hero %}
    <div class="hero-region py-5 bg-primary text-white">
      {{ page.hero }}
    </div>
  {% endif %}

  <main class="py-4">
    <div class="row">
      {% if page.sidebar_first %}
        <aside class="col-md-3">{{ page.sidebar_first }}</aside>
      {% endif %}

      <div class="{{ sidebar_first_classes|default('col-md-9') }}">
        {{ page.content }}
      </div>
    </div>
  </main>

  <footer class="site-footer py-4 bg-dark text-white">
    <div class="row">
      <div class="col-md-4">{{ page.footer_first }}</div>
      <div class="col-md-4">{{ page.footer_second }}</div>
      <div class="col-md-4">{{ page.footer_third }}</div>
    </div>
  </footer>
</div>
```

---

## Surcharger les Templates Bootstrap5

Le thème contrib `bootstrap5` fournit des templates qui utilisent les classes BS5. Pour les surcharger, **copier** le template depuis `web/themes/contrib/bootstrap5/templates/` vers `web/themes/custom/montheme/templates/` puis modifier.

### Template nœud — Card Bootstrap

```twig
{# templates/content/node--article--teaser.html.twig #}
<article{{ attributes.addClass('card', 'h-100', 'shadow-sm') }}>
  {% if content.field_image %}
    <div class="card-img-top overflow-hidden">
      {{ content.field_image }}
    </div>
  {% endif %}

  <div class="card-body d-flex flex-column">
    {{ title_prefix }}
    {% if not page %}
      <h3{{ title_attributes.addClass('card-title h5') }}>
        <a href="{{ url }}" rel="bookmark">{{ label }}</a>
      </h3>
    {% endif %}
    {{ title_suffix }}

    <div class="card-text flex-grow-1">
      {{ content|without('field_image', 'links') }}
    </div>

    {% if content.links %}
      <div class="card-footer bg-transparent border-0 px-0 mt-2">
        {{ content.links }}
      </div>
    {% endif %}
  </div>
</article>
```

### Template field — Wrapping Bootstrap

```twig
{# templates/field/field--field-tags.html.twig #}
{% if items %}
  <div class="field-tags d-flex flex-wrap gap-2 mt-2">
    {% for item in items %}
      <span class="badge bg-secondary">{{ item.content }}</span>
    {% endfor %}
  </div>
{% endif %}
```

### Template block — Card Bootstrap

```twig
{# templates/block/block--bundle--promo.html.twig #}
<div{{ attributes.addClass('block', 'card', 'border-0', 'bg-light') }}>
  {{ title_prefix }}
  {% if label %}
    <div class="card-header">
      <h2{{ title_attributes.addClass('h5', 'mb-0') }}>{{ label }}</h2>
    </div>
  {% endif %}
  {{ title_suffix }}

  <div class="card-body">
    {{ content }}
  </div>
</div>
```

### Template views — Grille Bootstrap

```twig
{# templates/views/views-view-unformatted--catalogue--page.html.twig #}
{% if title %}
  {{ title }}
{% endif %}

<div class="row row-cols-1 row-cols-sm-2 row-cols-md-3 g-4">
  {% for row in rows %}
    <div class="col">
      {{ row.content }}
    </div>
  {% endfor %}
</div>
```

---

## Menu de Navigation Responsive

Le thème Bootstrap5 gère le menu via son propre template. Pour personnaliser :

```twig
{# templates/navigation/menu--main.html.twig #}
{% import _self as menus %}

{#
  Macros pour le menu récursif Bootstrap 5.
  Supporte les dropdowns pour les sous-menus.
#}
{% macro menu_links(items, attributes, menu_level) %}
  {% if items %}
    {% if menu_level == 0 %}
      <ul class="navbar-nav me-auto mb-2 mb-lg-0">
    {% else %}
      <ul class="dropdown-menu">
    {% endif %}
    {% for item in items %}
      {% if menu_level == 0 %}
        <li class="nav-item{{ item.below ? ' dropdown' : '' }}">
          <a
            href="{{ item.url }}"
            class="nav-link{{ item.in_active_trail ? ' active' : '' }}{{ item.below ? ' dropdown-toggle' : '' }}"
            {{ item.below ? 'data-bs-toggle="dropdown" aria-expanded="false"' : '' }}
          >{{ item.title }}</a>
          {% if item.below %}
            {{ _self.menu_links(item.below, attributes, menu_level + 1) }}
          {% endif %}
        </li>
      {% else %}
        <li>
          <a href="{{ item.url }}" class="dropdown-item{{ item.in_active_trail ? ' active' : '' }}">
            {{ item.title }}
          </a>
        </li>
      {% endif %}
    {% endfor %}
    </ul>
  {% endif %}
{% endmacro %}

<nav class="navbar navbar-expand-lg navbar-light bg-white shadow-sm">
  <div class="container">
    <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#mainNav">
      <span class="navbar-toggler-icon"></span>
    </button>
    <div class="collapse navbar-collapse" id="mainNav">
      {{ _self.menu_links(items, attributes, 0) }}
    </div>
  </div>
</nav>
```

---

## Formulaires avec Classes Bootstrap

Le thème contrib `bootstrap5` applique automatiquement les classes Bootstrap via ses templates. Pour personnaliser davantage :

```php
// Dans montheme.theme — ajouter les classes Bootstrap aux éléments de formulaire
function montheme_preprocess_input(&$variables) {
  $type = $variables['element']['#type'] ?? '';
  $ignore_types = ['submit', 'button', 'hidden', 'checkbox', 'radio', 'image'];

  if (!in_array($type, $ignore_types)) {
    $variables['attributes']['class'][] = 'form-control';
  }
}

function montheme_preprocess_select(&$variables) {
  $variables['attributes']['class'][] = 'form-select';
}

function montheme_preprocess_textarea(&$variables) {
  $variables['attributes']['class'][] = 'form-control';
}
```

```twig
{# templates/form/input--submit.html.twig #}
<input{{ attributes.addClass('btn', 'btn-primary') }}>
```

---

## Carousel pour Paragraphs Media

```twig
{# templates/paragraphs/paragraph--carousel.html.twig #}
{% set carousel_id = 'carousel-' ~ paragraph.id() %}

<div id="{{ carousel_id }}" class="carousel slide" data-bs-ride="carousel">
  {# Indicateurs #}
  <div class="carousel-indicators">
    {% for key, item in content.field_slides['#items'] %}
      <button type="button"
        data-bs-target="#{{ carousel_id }}"
        data-bs-slide-to="{{ key }}"
        class="{{ key == 0 ? 'active' : '' }}"
      ></button>
    {% endfor %}
  </div>

  {# Slides #}
  <div class="carousel-inner">
    {% for key, slide in content.field_slides %}
      {% if slide['#paragraph'] is defined %}
        <div class="carousel-item{{ key == 0 ? ' active' : '' }}">
          {{ slide }}
        </div>
      {% endif %}
    {% endfor %}
  </div>

  {# Contrôles #}
  <button class="carousel-control-prev" type="button" data-bs-target="#{{ carousel_id }}" data-bs-slide="prev">
    <span class="carousel-control-prev-icon"></span>
    <span class="visually-hidden">Précédent</span>
  </button>
  <button class="carousel-control-next" type="button" data-bs-target="#{{ carousel_id }}" data-bs-slide="next">
    <span class="carousel-control-next-icon"></span>
    <span class="visually-hidden">Suivant</span>
  </button>
</div>
```

---

## SCSS Bootstrap — Compiler depuis le Sous-Thème

### Option A — Bootstrap via npm (recommandé pour les projets avec build pipeline)

```scss
// src/scss/main.scss

// 1. Surcharger les variables Bootstrap AVANT l'import
$primary: #2d6a4f;
$secondary: #52b788;
$font-family-base: 'Inter', sans-serif;
$border-radius: 0.5rem;
$spacer: 1rem;

// Surcharges de la grille Bootstrap
$grid-columns: 12;
$grid-gutter-width: 1.5rem;
$container-max-widths: (
  sm: 540px,
  md: 720px,
  lg: 960px,
  xl: 1140px,
  xxl: 1320px
);

// 2. Importer Bootstrap (depuis node_modules)
@import "bootstrap/scss/bootstrap";

// 3. Styles custom du thème par-dessus Bootstrap
@import "base/typography";
@import "base/variables-custom";
@import "layout/header";
@import "layout/footer";
@import "components/navigation";
@import "components/cards";
@import "components/forms";
@import "theme/colors";
```

### Option B — Bootstrap via CDN (sans build pipeline)

```yaml
# montheme.libraries.yml
bootstrap5-cdn:
  version: 5.3.2
  css:
    theme:
      https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css:
        { type: external, minified: true }
  js:
    https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/js/bootstrap.bundle.min.js:
      { type: external, minified: true }
```

```yaml
# montheme.info.yml — désactiver le CSS du thème contrib et utiliser CDN
libraries-override:
  bootstrap5/bootstrap5: false

libraries:
  - montheme/bootstrap5-cdn
  - montheme/global-styling
```

---

## Surcharger les Variables SCSS Bootstrap

```scss
// src/scss/base/_variables-override.scss
// Toujours définir ces variables AVANT @import "bootstrap/scss/bootstrap"

// Couleurs
$blue:    #0d6efd;
$primary: #1a5276;          // Couleur primaire du projet
$secondary: #7f8c8d;
$success: #27ae60;
$danger:  #e74c3c;
$warning: #f39c12;

// Typographie
$font-family-sans-serif: 'Roboto', -apple-system, sans-serif;
$font-size-base:          1rem;
$line-height-base:        1.6;
$headings-font-weight:    700;

// Espacements
$spacer: 1rem;
$spacers: (
  0: 0,
  1: $spacer * 0.25,
  2: $spacer * 0.5,
  3: $spacer,
  4: $spacer * 1.5,
  5: $spacer * 3,
  6: $spacer * 4,    // Spacer custom ajouté
);

// Boutons
$btn-border-radius:    0.375rem;
$btn-padding-x:        1.25rem;
$btn-padding-y:        0.5rem;
$btn-font-weight:      600;

// Cards
$card-border-radius:   0.75rem;
$card-box-shadow:      0 2px 8px rgba(0,0,0,0.08);

// Navbar
$navbar-padding-y:     1rem;
$navbar-light-color:   rgba(0,0,0,0.75);
```

---

## Anti-Patterns Bootstrap5 + Drupal

| Anti-pattern | Problème | Solution |
|-------------|---------|---------|
| Modifier directement `bootstrap5/templates/` | Écrasé à chaque mise à jour du contrib | Copier dans `montheme/templates/` et modifier là |
| Modifier `bootstrap5/css/bootstrap5.css` | Idem — écrasé à la mise à jour | SCSS dans le sous-thème, ou `libraries-override` |
| `base theme: false` avec Bootstrap5 | Perd tous les templates BS5 | Toujours `base theme: bootstrap5` dans le sous-thème |
| Charger Bootstrap via CDN ET via le thème contrib | Double chargement, conflits | Choisir l'un ou l'autre, désactiver l'autre |
| `@import url("https://...")` dans le SCSS | Bloque le rendu (render-blocking) | Charger via `.libraries.yml` avec `type: external` |
| Classes Bootstrap inline dans le PHP | Mélange CSS et logique | Templates Twig pour la présentation |
| `jQuery` pour les composants BS5 | BS5 n'a plus besoin de jQuery | Utiliser l'API JS native de Bootstrap 5 |
| Inclure `node_modules/bootstrap` dans git | Répertoire énorme (~50MB+) | `.gitignore` + `npm install` dans le process de build |
