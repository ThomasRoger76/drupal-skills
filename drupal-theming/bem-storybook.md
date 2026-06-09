# BEM & Storybook — Composants Drupal

## PARTIE A — BEM (Block Element Modifier) pour les thèmes Drupal

### Rappel BEM

BEM est une convention de nommage CSS qui découpe chaque composant en trois niveaux :

```
.block {}                    → Composant autonome (ex: card, menu, hero)
.block__element {}           → Partie interne du composant (ex: card__title)
.block--modifier {}          → Variante du composant entier (ex: card--featured)
.block__element--modifier {} → Variante d'une partie (ex: card__title--large)
```

**Règle d'or :** un bloc BEM doit pouvoir être déplacé n'importe où dans la page sans casser son style. Il ne dépend jamais de son contexte parent.

---

### BEM dans un template Twig Drupal

```twig
{# templates/node/node--article--teaser.html.twig #}
{# block = card — indépendant du type de nœud Drupal #}

<article {{ attributes.addClass('card', 'card--' ~ node.bundle) }}>

  {# element = card__image — partie du composant #}
  {% if content.field_image is not empty %}
    <div class="card__image">
      {{ content.field_image }}
    </div>
  {% endif %}

  {# element = card__body #}
  <div class="card__body">

    <h2 class="card__title">
      <a class="card__link" href="{{ url }}">{{ label }}</a>
    </h2>

    {# modifier selon le bundle — variante de l'élément #}
    <div class="card__text card__text--{{ node.bundle }}">
      {{ content.body }}
    </div>

    {# élément répété : card__tag #}
    {% if content.field_tags is not empty %}
      <div class="card__tags">
        {% for tag in content.field_tags %}
          <span class="card__tag">{{ tag }}</span>
        {% endfor %}
      </div>
    {% endif %}

  </div>

  {# element card__footer avec modifier --centered #}
  <footer class="card__footer card__footer--centered">
    <time class="card__date">{{ node.created.value|date('d/m/Y') }}</time>
  </footer>

</article>
```

---

### BEM dans le preprocess PHP

```php
<?php
// mon_theme.theme

/**
 * Implémente hook_preprocess_node() pour ajouter les classes BEM.
 */
function mon_theme_preprocess_node(array &$variables): void {
  $node = $variables['node'];

  // Classes BEM sur l'élément racine via attributes
  $variables['attributes']['class'][] = 'card';
  $variables['attributes']['class'][] = 'card--' . $node->bundle();

  // Modifiers conditionnels
  if ($node->isSticky()) {
    $variables['attributes']['class'][] = 'card--sticky';
  }

  if (!$node->isPublished()) {
    $variables['attributes']['class'][] = 'card--draft';
  }

  // Modifier selon le view_mode courant
  $view_mode = $variables['view_mode'] ?? 'default';
  if ($view_mode !== 'default') {
    $variables['attributes']['class'][] = 'card--' . str_replace('_', '-', $view_mode);
  }
}

/**
 * Ajouter des classes BEM sur un champ spécifique.
 */
function mon_theme_preprocess_field(array &$variables): void {
  $field_name = $variables['field_name'];
  $bundle     = $variables['element']['#bundle'] ?? '';

  // Transformer les noms de champs Drupal en éléments BEM propres
  // field_image → card__image
  $bem_element = str_replace(['field_', '_'], ['', '-'], $field_name);
  $variables['attributes']['class'][] = 'card__' . $bem_element;
}
```

---

### BEM dans le CSS/SCSS

```scss
// components/_card.scss
// Convention SCSS : nesting + & pour les éléments et modifiers

.card {
  // block — styles de base
  padding: var(--spacing-md);
  border-radius: var(--border-radius);
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  overflow: hidden;

  // ─── Éléments ──────────────────────────────────────────────────────────────

  &__image {
    aspect-ratio: 16 / 9;
    overflow: hidden;

    img {
      width: 100%;
      height: 100%;
      object-fit: cover;
      transition: transform 0.3s ease;
    }
  }

  &__body {
    padding: var(--spacing-md);
  }

  &__title {
    font-size: var(--font-size-lg);
    font-weight: 700;
    margin-bottom: var(--spacing-sm);
    line-height: 1.3;
  }

  &__link {
    color: inherit;
    text-decoration: none;

    &:hover {
      color: var(--color-primary);
    }
  }

  &__text {
    color: var(--color-text-muted);
    font-size: var(--font-size-sm);
    line-height: 1.6;

    // modifier sur l'élément — variante par bundle
    &--article {
      max-height: 4.8em;     // 3 lignes
      overflow: hidden;
    }

    &--page {
      max-height: none;      // Pages — texte complet
    }
  }

  &__tags {
    display: flex;
    flex-wrap: wrap;
    gap: var(--spacing-xs);
    margin-top: var(--spacing-sm);
  }

  &__tag {
    padding: 2px var(--spacing-xs);
    background: var(--color-primary-light);
    color: var(--color-primary-dark);
    border-radius: 999px;
    font-size: var(--font-size-xs);
    font-weight: 500;
  }

  &__footer {
    padding: var(--spacing-sm) var(--spacing-md);
    border-top: 1px solid var(--color-border);

    // modifier --centered
    &--centered {
      display: flex;
      justify-content: center;
    }
  }

  &__date {
    font-size: var(--font-size-xs);
    color: var(--color-text-muted);
  }

  // ─── Modifiers du bloc ─────────────────────────────────────────────────────

  &--featured {
    border: 2px solid var(--color-primary);
    background: var(--color-primary-light);

    .card__title {
      color: var(--color-primary-dark);
    }
  }

  &--sticky {
    position: relative;

    &::before {
      content: 'Épinglé';
      position: absolute;
      top: var(--spacing-sm);
      right: var(--spacing-sm);
      background: var(--color-warning);
      color: white;
      font-size: var(--font-size-xs);
      padding: 2px 8px;
      border-radius: 4px;
    }
  }

  &--draft {
    opacity: 0.7;
    border-style: dashed;
  }

  &--teaser {
    .card__image:hover img {
      transform: scale(1.05);
    }
  }
}
```

---

### BEM avec SDC (Single Directory Components, D11)

```yaml
# web/themes/custom/mon_theme/components/card/card.component.yml
name: Card
description: 'Carte de contenu réutilisable — articles, pages, produits.'
props:
  type: object
  properties:
    modifier:
      type: string
      title: 'Variante'
      description: 'Modifier BEM appliqué au bloc card'
      enum: [default, featured, compact, sticky]
      default: default
    title:
      type: string
      title: 'Titre'
    body:
      type: string
      title: 'Corps du texte'
    image_url:
      type: string
      title: 'URL de l''image'
    date:
      type: string
      title: 'Date formatée'
  required: [title]
slots:
  tags:
    title: 'Tags'
    description: 'Liste des tags du contenu'
```

```twig
{# web/themes/custom/mon_theme/components/card/card.twig — SDC : {id}.twig, jamais .html.twig #}
<article class="card card--{{ modifier|default('default') }}">
  {% if image_url %}
    <div class="card__image">
      <img src="{{ image_url }}" alt="{{ title }}" loading="lazy">
    </div>
  {% endif %}
  <div class="card__body">
    <h2 class="card__title">{{ title }}</h2>
    {% if body %}
      <div class="card__text">{{ body }}</div>
    {% endif %}
    {% if tags %}
      <div class="card__tags">{{ tags }}</div>
    {% endif %}
  </div>
  {% if date %}
    <footer class="card__footer">
      <time class="card__date">{{ date }}</time>
    </footer>
  {% endif %}
</article>
```

---

### Règles BEM pour Drupal — Anti-patterns

```scss
// ❌ Cibler les classes générées automatiquement par Drupal
.node--article .field-image { }
// → Fragile : les classes Drupal changent selon la config, le view_mode, le thème de base

// ✅ Ajouter une classe BEM contrôlée via preprocess ou addClass()
.card__image { }
```

```twig
{# ❌ Cibler .field-body dans un template parent #}
{{ content.body }}
{# Le div wrappeur porte .field--name-body.field--type-text-with-summary... #}
{# → Ne jamais styler ces classes dans votre SCSS — elles sont générées #}

{# ✅ Wrapper explicite avec classe BEM #}
<div class="card__text">
  {{ content.body }}
</div>
```

```scss
// ❌ État global hors contexte BEM
.is-active { color: var(--color-primary); }
// → Manque de contexte : .is-active sur quoi ?

// ✅ État sur le bon composant
.menu__item--active { color: var(--color-primary); }
.card--active { border-color: var(--color-primary); }
```

```scss
// ❌ Nesting profond qui crée des sélecteurs trop spécifiques
.card {
  .card__body {
    .card__title {
      a { }   // Génère .card .card__body .card__title a — trop spécifique
    }
  }
}

// ✅ Un niveau de nesting maximum, toujours via &
.card {
  &__title {
    a { }   // .card__title a — correct
  }
}
```

---

## PARTIE B — Storybook + SDC (D11)

### Installation Storybook dans un thème Drupal

```bash
# Se placer dans le répertoire du thème custom
cd web/themes/custom/mon_theme

# Initialiser npm si pas encore fait
npm init -y

# Installer Storybook HTML (compatible avec les templates Drupal non-React)
npm install --save-dev @storybook/html @storybook/addon-essentials

# Initialisation interactive — choisir "HTML" comme framework
npx storybook@latest init --type html

# Lancer Storybook
npm run storybook
```

**Structure résultante :**
```
mon_theme/
├── .storybook/
│   ├── main.js          ← Configuration Storybook (addons, stories glob)
│   └── preview.js       ← Imports CSS globaux, backgrounds, decorators
├── components/
│   └── card/
│       ├── card.twig               ← Template SDC ({id}.twig, jamais .html.twig)
│       ├── card.component.yml      ← Schéma SDC
│       ├── card.css                ← Styles du composant
│       └── card.stories.js         ← Stories Storybook
└── package.json
```

---

### Story pour un SDC — Exemple complet

```javascript
// components/card/card.stories.js

// Fonction helper pour simuler le rendu du template (sans Twig en JS)
const renderCard = ({ modifier, title, body, image_url, date, tags }) => `
  <article class="card card--${modifier}">
    ${image_url ? `
      <div class="card__image">
        <img src="${image_url}" alt="${title}" loading="lazy">
      </div>` : ''}
    <div class="card__body">
      <h2 class="card__title">${title}</h2>
      ${body ? `<div class="card__text card__text--article">${body}</div>` : ''}
      ${tags?.length ? `
        <div class="card__tags">
          ${tags.map(tag => `<span class="card__tag">${tag}</span>`).join('')}
        </div>` : ''}
    </div>
    ${date ? `
      <footer class="card__footer card__footer--centered">
        <time class="card__date">${date}</time>
      </footer>` : ''}
  </article>
`;

export default {
  title: 'Composants/Card',
  tags: ['autodocs'],   // Génère une page de documentation automatique
  render: renderCard,
  argTypes: {
    modifier: {
      control: { type: 'select' },
      options: ['default', 'featured', 'compact', 'sticky'],
      description: 'Modifier BEM — variante visuelle du composant',
    },
    title: {
      control: 'text',
      description: 'Titre de la carte',
    },
    body: {
      control: 'text',
      description: 'Corps du texte (optionnel)',
    },
    image_url: {
      control: 'text',
      description: 'URL de l\'image (optionnel)',
    },
    date: {
      control: 'text',
      description: 'Date formatée (optionnel)',
    },
  },
  args: {
    modifier: 'default',
    title: 'Titre de l\'article exemple',
    body: 'Description courte du contenu pour illustrer le rendu de la carte dans son état par défaut.',
    image_url: 'https://picsum.photos/seed/drupal/800/450',
    date: '15 mai 2026',
  },
};

// ─── Stories ──────────────────────────────────────────────────────────────────

export const Default = {};

export const Featured = {
  args: {
    ...Default.args,
    modifier: 'featured',
    title: 'Article mis en avant',
  },
};

export const Compact = {
  args: {
    ...Default.args,
    modifier: 'compact',
    image_url: null,
    body: 'Résumé court.',
  },
};

export const SansImage = {
  name: 'Sans image',
  args: {
    ...Default.args,
    image_url: null,
  },
};

export const AvecTags = {
  name: 'Avec tags',
  args: {
    ...Default.args,
    tags: ['Drupal', 'Theming', 'BEM'],
  },
};

export const Brouillon = {
  name: 'État brouillon (draft)',
  args: {
    ...Default.args,
    modifier: 'draft',
    title: '[BROUILLON] Contenu en cours de rédaction',
    image_url: null,
  },
};
```

---

### Intégration CSS du thème dans Storybook

```javascript
// .storybook/preview.js

// Importer le CSS compilé du thème pour que les stories reflètent le vrai rendu
import '../css/main.css';         // CSS global du thème (variables, reset, typographie)
import '../css/components.css';   // Tous les composants compilés
// Ou composant par composant si le build est granulaire :
// import '../components/card/card.css';

/** @type { import('@storybook/html').Preview } */
const preview = {
  parameters: {
    backgrounds: {
      default: 'light',
      values: [
        { name: 'light',       value: '#ffffff' },
        { name: 'grey',        value: '#f5f5f5' },
        { name: 'dark',        value: '#1a1a2e' },
        { name: 'primary-bg',  value: '#eef4ff' },
      ],
    },
    // Afficher les docs en français
    docs: {
      autodocs: 'tag',
    },
    // Viewport presets — tester mobile first
    viewport: {
      viewports: {
        mobile:  { name: 'Mobile',  styles: { width: '375px',  height: '667px' } },
        tablet:  { name: 'Tablette', styles: { width: '768px',  height: '1024px' } },
        desktop: { name: 'Desktop', styles: { width: '1280px', height: '900px' } },
      },
      defaultViewport: 'desktop',
    },
  },
};

export default preview;
```

---

### Configuration `.storybook/main.js`

```javascript
// .storybook/main.js
/** @type { import('@storybook/html-vite').StorybookConfig } */
const config = {
  // Chercher les stories dans tous les sous-répertoires du thème
  stories: [
    '../components/**/*.stories.@(js|mjs)',
    '../templates/**/*.stories.@(js|mjs)',
  ],
  addons: [
    '@storybook/addon-essentials',   // Controls, Actions, Docs, Backgrounds, Viewport
    '@storybook/addon-a11y',          // Audit accessibilité WCAG dans chaque story
  ],
  framework: {
    name: '@storybook/html-vite',
    options: {},
  },
  docs: {
    autodocs: 'tag',
  },
};

export default config;
```

---

### Scripts npm pour Storybook

```json
// package.json
{
  "name": "mon-theme",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "storybook":       "storybook dev -p 6006",
    "build-storybook": "storybook build --output-dir ../../sites/default/files/storybook",
    "watch":           "sass --watch scss:css --style=compressed",
    "build":           "sass scss:css --style=compressed --no-source-map"
  },
  "devDependencies": {
    "@storybook/addon-a11y": "^8.0.0",
    "@storybook/addon-essentials": "^8.0.0",
    "@storybook/html-vite": "^8.0.0",
    "sass": "^1.77.0",
    "storybook": "^8.0.0"
  }
}
```

---

### Service Docker Compose — Storybook parallèle au thème

```yaml
# docker-compose.yml — ajout du service Storybook

services:
  # ... autres services (php, mariadb, caddy) ...

  theme:
    image: node:22-alpine
    working_dir: /app/web/themes/custom/mon_theme
    command: npm run watch     # Compilation SCSS en continu
    volumes:
      - .:/app
    restart: unless-stopped

  storybook:
    image: node:22-alpine
    working_dir: /app/web/themes/custom/mon_theme
    command: sh -c "npm install && npm run storybook"
    ports:
      - "${STORYBOOK_PORT:-6006}:6006"
    volumes:
      - .:/app
    environment:
      - NODE_ENV=development
    depends_on:
      - theme    # S'assure que npm install a eu lieu via le service theme
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:6006"]
      interval: 30s
      timeout: 10s
      retries: 3
```

```bash
# Démarrer uniquement Storybook
docker compose up storybook -d

# Accès dans le navigateur
open http://localhost:6006

# Voir les logs Storybook
docker compose logs -f storybook

# Rebuild si les dépendances changent
docker compose exec storybook npm install
docker compose restart storybook
```

---

### Anti-patterns Storybook/BEM

```javascript
// ❌ Dupliquer tout le HTML Drupal dans les stories
// → Les stories deviennent un second source of truth divergeant

// ✅ Les stories reproduisent uniquement la structure BEM du composant
// L'HTML exact (attributs Drupal, classes système) reste dans le template Twig
```

```scss
// ❌ Surspécifier pour "gagner" contre les classes Drupal
.node--article .field--name-field-image .card__image { }

// ✅ Concevoir le composant pour fonctionner sans dépendre des classes Drupal
// Ajouter les classes BEM via preprocess, pas via des surspécifications
.card__image { }
```

```javascript
// ❌ Une seule story "Default" sans variantes
export const Default = { args: { title: 'Test' } };

// ✅ Couvrir tous les états et modifiers dans des stories séparées
// → Permet de voir les régressions visuelles d'un coup d'œil
export const Default   = { ... };
export const Featured  = { ... };
export const SansImage = { ... };
export const Mobile    = { parameters: { viewport: { defaultViewport: 'mobile' } } };
```

```bash
# ❌ Checker le build Storybook en production sans tester l'accessibilité
# ✅ Installer @storybook/addon-a11y pour auditer chaque story
npm install --save-dev @storybook/addon-a11y
# → Ajouter dans .storybook/main.js addons[]
# → Chaque story affiche un panel "Accessibility" avec les violations WCAG
```
