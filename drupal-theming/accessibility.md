---
name: drupal-theming — accessibility WCAG
description: Implémenter l'accessibilité WCAG 2.1 AA dans les thèmes Drupal - ARIA, navigation clavier, contraste, images, formulaires, et tests automatisés.
---

# Accessibilité WCAG 2.1 AA — Référence Drupal

## Pourquoi c'est obligatoire

```
Légal (France) :
  → RGAA (Référentiel Général d'Amélioration de l'Accessibilité)
  → Obligatoire pour les organismes publics et grandes entreprises
  → Amende potentielle + risque d'image

WCAG 2.1 AA :
  → Standard international reconnu
  → 4 principes : Perceptible, Utilisable, Compréhensible, Robuste
  → Niveau A (minimum) → AA (recommandé) → AAA (optimal)
```

---

## Images — Alt Text Obligatoire

```twig
{# ✅ Image informative — alt descriptif #}
<img src="{{ image_url }}" alt="{{ image_alt }}" width="{{ width }}" height="{{ height }}">

{# ✅ Image décorative — alt vide (pas absent !) #}
<img src="{{ decorative_img }}" alt="" role="presentation">

{# ✅ Image complexe (graphique, infographie) — alt + longdesc ou figure+figcaption #}
<figure>
  <img src="{{ chart.url }}" alt="Graphique : ventes 2026 par trimestre">
  <figcaption>Q1: 1200 — Q2: 1450 — Q3: 1380 — Q4: 1620 unités</figcaption>
</figure>

{# ❌ WCAG A1.1 — Image sans alt = erreur critique #}
<img src="{{ url }}">

{# ❌ WCAG A1.1 — Alt "image" ou filename = inutile #}
<img src="{{ url }}" alt="image" >
<img src="{{ url }}" alt="photo_2026.jpg">
```

```php
// Dans le champ Image — rendre l'alt obligatoire
// config/install/field.field.node.article.field_image.yml
settings:
  alt_field: true
  alt_field_required: true   # ← OBLIGATOIRE
  title_field: false
```

---

## Navigation Clavier

```twig
{# Tous les éléments interactifs doivent être accessibles au clavier #}

{# ✅ Lien skip-to-content (premier élément de la page) #}
<a href="#main-content" class="skip-link visually-hidden focusable">
  {{ 'Aller au contenu principal'|t }}
</a>

{# ✅ Focus visible — toujours visible, ne pas supprimer #}
{# Ne jamais mettre outline: none dans le CSS sans alternative #}

{# ✅ Ordre de focus logique (tabindex=0 pour éléments custom interactifs) #}
<div role="button" tabindex="0"
     onkeydown="if(event.key==='Enter'||event.key===' ')this.click()"
     onclick="toggleMenu()">
  Menu
</div>

{# ✅ Focus trap dans les modales #}
{# (voir drupal/dialog_off_canvas ou pattern focus-trap) #}
```

```css
/* ✅ Styles focus visibles et contrastés */
:focus-visible {
  outline: 3px solid #005fcc;
  outline-offset: 2px;
  border-radius: 2px;
}

/* ✅ Skip link — visible seulement au focus */
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  background: #000;
  color: #fff;
  padding: 8px;
  z-index: 9999;
  transition: top 0.2s;
}
.skip-link:focus {
  top: 0;
}
```

---

## ARIA — Roles et Attributs

```twig
{# Navigation — landmarks ARIA #}
<header role="banner">
  <nav role="navigation" aria-label="{{ 'Navigation principale'|t }}">
    ...
  </nav>
</header>

<main id="main-content" role="main" tabindex="-1">
  ...
</main>

<aside role="complementary" aria-label="{{ 'Informations complémentaires'|t }}">
  ...
</aside>

<footer role="contentinfo">
  ...
</footer>

{# Boutons et toggles #}
<button
  aria-expanded="{{ menu_open ? 'true' : 'false' }}"
  aria-controls="mobile-menu"
  aria-label="{{ 'Ouvrir le menu'|t }}">
  <span aria-hidden="true">☰</span>
</button>

{# Carousels — annonces live #}
<div
  role="region"
  aria-label="{{ 'Carrousel'|t }}"
  aria-live="polite"
  aria-atomic="false">
  ...
</div>

{# Messages d'état dynamiques #}
<div
  role="status"
  aria-live="polite"
  aria-atomic="true"
  class="visually-hidden">
  {{ status_message }}
</div>

{# Erreurs de formulaire #}
<input
  type="email"
  id="email"
  aria-required="true"
  aria-invalid="{{ has_error ? 'true' : 'false' }}"
  aria-describedby="{{ has_error ? 'email-error' : '' }}">
<p id="email-error" role="alert" class="{{ has_error ? '' : 'visually-hidden' }}">
  {{ email_error_message }}
</p>
```

---

## Contraste des Couleurs

```css
/* WCAG 2.1 AA — Ratios minimum :
   Texte normal (< 18pt) : 4.5:1
   Texte grand (≥ 18pt ou 14pt gras) : 3:1
   Composants UI (boutons, icônes) : 3:1

   Outils de vérification :
   - WebAIM Contrast Checker : webaim.org/resources/contrastchecker
   - browser DevTools → Accessibility tab
*/

/* ✅ Bon contraste : #1a5276 sur blanc = ratio 8.6:1 */
.text-primary { color: #1a5276; }

/* ❌ Mauvais contraste : #999 sur blanc = ratio 2.85:1 */
.text-muted { color: #999; } /* À corriger vers #767676 minimum (4.5:1) */
```

---

## Drupal Core — Aide à l'Accessibilité

```php
// Drupal fournit des helpers natifs
use Drupal\Component\Utility\Html;

// Générer un ID unique pour les associations ARIA
$id = Html::getUniqueId('my-element');
// → 'my-element', 'my-element--2', etc.

// Composant visually-hidden natif Drupal
$element['screen_reader_text'] = [
  '#type' => 'html_tag',
  '#tag' => 'span',
  '#attributes' => ['class' => ['visually-hidden']],
  '#value' => $this->t('Texte pour lecteurs d\'écran'),
];

// Messages avec rôle ARIA
\Drupal::messenger()->addStatus(
  $this->t('Formulaire enregistré.'),
  TRUE  // ← TRUE = accessible (role="status")
);
```

---

## Tester l'Accessibilité Drupal

```bash
# 1. Tests automatiques avec axe-core (CI/CD)
npm install --save-dev axe-playwright @playwright/test

# playwright.config.js → utiliser axe dans les tests
# npx playwright test --project=accessibility

# 2. Module Drupal accessibility_checker
composer require drupal/editoria11y
drush en editoria11y -y
# → Affiche un bilan d'accessibilité sur chaque page en développement

# 3. Audit rapide avec Lighthouse
npx lighthouse https://mon-site.com --only-categories=accessibility --output html

# 4. Test manuel avec NVDA (Windows) ou VoiceOver (Mac)
# → Naviguer au clavier seulement (Tab, Enter, Arrow keys)
# → Lire la page avec le lecteur d'écran
```

---

## Classe `visually-hidden` Standard

```css
/* Classe native Drupal — pour le contenu visible uniquement aux lecteurs d'écran */
.visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

/* Variante : visible au focus (pour les skip links) */
.visually-hidden.focusable:focus,
.visually-hidden.focusable:focus-within {
  position: static;
  width: auto;
  height: auto;
  overflow: visible;
  clip: auto;
  white-space: normal;
}
```
