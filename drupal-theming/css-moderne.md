# CSS Moderne — Drupal 11 / Navigateurs Modernes

## Vue d'ensemble

Techniques CSS modernes adaptées à Drupal 11 : Container Queries, Dark Mode, CSS Logical Properties, et Cascade Layers. Ces approches remplacent les anciennes méthodes basées uniquement sur les media queries viewport.

---

## 1. CSS Container Queries — Responsive selon le Conteneur

Les Container Queries permettent de styliser un composant en fonction de la taille de **son conteneur parent**, pas de la viewport. Idéal pour les composants réutilisables dans plusieurs régions Drupal.

```css
/* Déclarer le conteneur */
.card-wrapper {
  container-type: inline-size;
  container-name: card;
}

/* Styler selon la taille du conteneur */
@container card (min-width: 400px) {
  .card {
    display: grid;
    grid-template-columns: 200px 1fr;
  }
  .card__image {
    height: 100%;
  }
}

@container card (max-width: 399px) {
  .card {
    display: block;
  }
}
```

```twig
{# Template Drupal — déclarer le container wrapper #}
{# web/themes/custom/mon_theme/templates/content/node--article--card.html.twig #}
<div class="card-wrapper" style="container-type: inline-size; container-name: card;">
  <article{{ attributes.addClass('card') }}>
    {% if content.field_image %}
      <div class="card__image">
        {{ content.field_image }}
      </div>
    {% endif %}
    <div class="card__body">
      {{ title_prefix }}
      {% if not page %}
        <h3{{ title_attributes.addClass('card__title') }}>
          <a href="{{ url }}" rel="bookmark">{{ label }}</a>
        </h3>
      {% endif %}
      {{ title_suffix }}
      {{ content|without('field_image') }}
    </div>
  </article>
</div>
```

```css
/* Version propre — sans style inline, avec class CSS */
.card-wrapper {
  container-type: inline-size;
  container-name: card;
  width: 100%;
}

/* Dans le contexte d'une sidebar (conteneur étroit) */
@container card (max-width: 299px) {
  .card__title {
    font-size: var(--font-size-sm);
  }
  .card__image {
    display: none; /* Masquer l'image dans les petits conteneurs */
  }
}

/* Dans le contexte du contenu principal (conteneur large) */
@container card (min-width: 600px) {
  .card {
    display: grid;
    grid-template-columns: 300px 1fr;
    gap: 1.5rem;
    align-items: start;
  }
}
```

> **Support navigateurs :** Chrome 105+, Firefox 110+, Safari 16+. Pour les anciens navigateurs, définir un layout fallback sans `@container`.

---

## 2. Dark Mode — CSS Variables + prefers-color-scheme

```css
/* Variables CSS — thème clair/sombre */
:root {
  --color-bg: #ffffff;
  --color-text: #1a1a1a;
  --color-primary: #0071b8;
  --color-surface: #f5f5f5;
  --color-border: #e0e0e0;
  --color-shadow: rgba(0, 0, 0, 0.1);
}

@media (prefers-color-scheme: dark) {
  :root {
    --color-bg: #1a1a2e;
    --color-text: #e0e0e0;
    --color-primary: #4da6ff;
    --color-surface: #16213e;
    --color-border: #333355;
    --color-shadow: rgba(0, 0, 0, 0.4);
  }
}

/* Surcharge manuelle via data-theme (toggle utilisateur) */
[data-theme="dark"] {
  --color-bg: #1a1a2e;
  --color-text: #e0e0e0;
  --color-primary: #4da6ff;
  --color-surface: #16213e;
  --color-border: #333355;
  --color-shadow: rgba(0, 0, 0, 0.4);
}

[data-theme="light"] {
  --color-bg: #ffffff;
  --color-text: #1a1a1a;
  --color-primary: #0071b8;
  --color-surface: #f5f5f5;
  --color-border: #e0e0e0;
  --color-shadow: rgba(0, 0, 0, 0.1);
}

/* Utilisation des variables */
body {
  background-color: var(--color-bg);
  color: var(--color-text);
}

.card {
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  box-shadow: 0 2px 4px var(--color-shadow);
}

a {
  color: var(--color-primary);
}
```

```twig
{# Ajouter data-theme au html.html.twig #}
{# web/themes/custom/mon_theme/templates/html.html.twig #}
<!DOCTYPE html>
<html{{ html_attributes.setAttribute('data-theme', 'auto') }}>
  <head>
    <head-placeholder token="{{ placeholder_token }}">
    <title>{{ head_title|safe_join(' | ') }}</title>
    <css-placeholder token="{{ placeholder_token }}">
    <js-placeholder token="{{ placeholder_token }}">
  </head>
  <body{{ attributes }}>
    <a href="#main-content" class="visually-hidden focusable skip-link">
      {{ 'Skip to main content'|t }}
    </a>
    {{ page_top }}
    {{ page }}
    {{ page_bottom }}
    <js-bottom-placeholder token="{{ placeholder_token }}">
  </body>
</html>
```

```javascript
// web/themes/custom/mon_theme/js/dark-mode.js
// JS — permettre à l'utilisateur de forcer le mode
(function(Drupal, once) {
  Drupal.behaviors.darkModeToggle = {
    attach(context) {
      // Restaurer le thème sauvegardé au chargement
      const saved = localStorage.getItem('theme');
      if (saved) {
        document.documentElement.setAttribute('data-theme', saved);
      }

      once('dark-mode', '[data-dark-toggle]', context).forEach(btn => {
        // Mettre à jour le label du bouton selon l'état actuel
        const updateLabel = () => {
          const current = document.documentElement.getAttribute('data-theme');
          btn.setAttribute('aria-label',
            current === 'dark' ? Drupal.t('Activer le mode clair') : Drupal.t('Activer le mode sombre')
          );
          btn.setAttribute('aria-pressed', current === 'dark' ? 'true' : 'false');
        };

        updateLabel();

        btn.addEventListener('click', () => {
          const html = document.documentElement;
          const current = html.getAttribute('data-theme');
          const next = current === 'dark' ? 'light' : 'dark';
          html.setAttribute('data-theme', next);
          localStorage.setItem('theme', next);
          updateLabel();
        });
      });
    }
  };
})(Drupal, once);
```

```twig
{# Bouton toggle Dark Mode — à placer dans le header #}
<button
  data-dark-toggle
  aria-label="{{ 'Activer le mode sombre'|t }}"
  aria-pressed="false"
  class="dark-mode-toggle"
  type="button">
  <span class="dark-mode-toggle__icon dark-mode-toggle__icon--light">☀</span>
  <span class="dark-mode-toggle__icon dark-mode-toggle__icon--dark">☾</span>
  <span class="visually-hidden">{{ 'Basculer le thème'|t }}</span>
</button>
```

---

## 3. CSS Logical Properties — Support RTL/LTR Automatique

Les propriétés logiques s'adaptent automatiquement à la direction du texte (LTR/RTL). Indispensable sur les sites Drupal multilingues avec des langues RTL (arabe, hébreu, persan).

```css
/* ❌ Éviter en contexte multilingue (propriétés physiques) : */
.card {
  margin-left: 16px;
  padding-right: 24px;
  border-left: 2px solid var(--color-primary);
  text-align: left;
}

/* ✅ Utiliser les propriétés logiques : */
.card {
  margin-inline-start: 16px;         /* = margin-left en LTR, margin-right en RTL */
  padding-inline-end: 24px;          /* = padding-right en LTR, padding-left en RTL */
  border-inline-start: 2px solid var(--color-primary);
  text-align: start;                 /* = left en LTR, right en RTL */
}

/* Tableau complet des équivalences */
/* Axe inline (horizontal) */
/* margin-left       → margin-inline-start */
/* margin-right      → margin-inline-end */
/* padding-left      → padding-inline-start */
/* padding-right     → padding-inline-end */
/* border-left       → border-inline-start */
/* border-right      → border-inline-end */
/* left              → inset-inline-start */
/* right             → inset-inline-end */

/* Axe block (vertical) */
/* margin-top        → margin-block-start */
/* margin-bottom     → margin-block-end */
/* padding-top       → padding-block-start */
/* padding-bottom    → padding-block-end */

/* Raccourcis */
.element {
  margin-inline: 16px 24px;    /* inline-start inline-end */
  padding-block: 8px 12px;     /* block-start block-end */
  inset-inline: 0 auto;        /* left: 0; right: auto en LTR */
}
```

```twig
{# Drupal gère la direction via l'attribut dir sur <html> #}
{# Pour forcer la direction dans un composant : #}
<div class="rtl-aware-component" dir="{{ language.direction }}">
  <span class="icon" aria-hidden="true">→</span>
  <span class="label">{{ item.label }}</span>
</div>
```

```php
// preprocess — ajouter la direction dans les variables
function mon_theme_preprocess_html(array &$variables): void {
  $language = \Drupal::languageManager()->getCurrentLanguage();
  $variables['html_attributes']['dir'] = $language->getDirection();
  $variables['html_attributes']['lang'] = $language->getId();

  // Cache context obligatoire
  $variables['#cache']['contexts'][] = 'languages:language_interface';
}
```

---

## 4. CSS Cascade Layers — Organisation Avancée

Les Cascade Layers (`@layer`) permettent de contrôler explicitement la priorité CSS. Évite les guerres de spécificité entre les styles Drupal core, contrib et custom.

```css
/* web/themes/custom/mon_theme/css/base/layers.css */

/* Déclarer l'ordre des layers en premier — la déclaration détermine la priorité */
/* Un layer déclaré PLUS TARD est PLUS PRIORITAIRE en cas de conflit */
@layer reset, base, drupal, components, utilities, overrides;

/* Reset minimal */
@layer reset {
  *, *::before, *::after {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }
}

/* Styles de base (typographie, couleurs) */
@layer base {
  body {
    font-family: system-ui, -apple-system, sans-serif;
    font-size: 1rem;
    line-height: 1.6;
    background-color: var(--color-bg);
    color: var(--color-text);
  }

  h1, h2, h3, h4, h5, h6 {
    line-height: 1.25;
  }
}

/* Surcharges spécifiques des styles Drupal core */
@layer drupal {
  /* Neutraliser les styles de messages Drupal pour les adapter au thème */
  .messages {
    border-radius: 4px;
    padding: 1rem;
  }
}

/* Composants réutilisables */
@layer components {
  .card {
    background: var(--color-surface);
    border: 1px solid var(--color-border);
    border-radius: 8px;
    overflow: hidden;
  }

  .card__title {
    font-size: 1.25rem;
    font-weight: 600;
    margin-block-end: 0.5rem;
  }

  .btn {
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.5rem 1rem;
    background: var(--color-primary);
    color: #fff;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    text-decoration: none;
  }
}

/* Utilitaires (haute priorité — remplacent les composants) */
@layer utilities {
  .visually-hidden {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0 0 0 0);
    white-space: nowrap;
    border: 0;
  }

  .sr-only { /* Alias moderne */
    position: absolute;
    width: 1px;
    height: 1px;
    overflow: hidden;
    clip: rect(0 0 0 0);
  }

  .text-center { text-align: center; }
  .hidden { display: none; }
}

/* Surcharges d'urgence — priorité maximale */
@layer overrides {
  /* Réservé aux corrections ciblées sur des modules contrib récalcitrants */
  /* Utiliser avec parcimonie */
}
```

```yaml
# mon_theme.libraries.yml — charger le fichier layers en premier
global-styles:
  css:
    base:
      css/base/layers.css: {}     # Layers en premier (ordre de déclaration)
      css/base/variables.css: {}  # Variables CSS
      css/base/typography.css: {}
    component:
      css/components/card.css: {}
      css/components/button.css: {}
    theme:
      css/theme/overrides.css: {}
```

> **Anti-pattern :** mélanger des styles avec et sans `@layer` dans le même projet. Les styles sans `@layer` sont toujours PLUS PRIORITAIRES que ceux dans un layer.

---

## Référence Rapide

| Technique | Cas d'usage Drupal | Support |
|-----------|-------------------|---------|
| Container Queries | Composants réutilisables (card, teaser) | Chrome 105+, FF 110+, Safari 16+ |
| CSS Variables + dark mode | Thème clair/sombre, tokens de design | Tous navigateurs modernes |
| Logical Properties | Sites multilingues avec RTL | Chrome 89+, FF 68+, Safari 15+ |
| Cascade Layers | Organisation CSS multi-sources | Chrome 99+, FF 97+, Safari 15.4+ |

**Voir aussi :**
- [preprocess.md](preprocess.md) — Ajouter des classes CSS depuis PHP
- [libraries-assets.md](libraries-assets.md) — Charger des fichiers CSS conditionnellement
- [twig3-accessibility.md](twig3-accessibility.md) — Accessibilité et direction du texte
