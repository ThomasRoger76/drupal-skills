---
name: drupal-sdc — storybook
description: Documenter et tester les SDC Drupal dans Storybook - configuration, stories HTML, mock data, et intégration avec le workflow de développement.
---

# SDC & Storybook — Documentation Visuelle

## Pourquoi Storybook avec SDC

```
SDC sans Storybook :
  → Tester un composant = créer un nœud Drupal de test
  → Impossible de voir toutes les variantes sans données réelles
  → Pas de documentation visuelle pour les équipes design/frontend

SDC + Storybook :
  ✅ Voir chaque composant isolé, avec toutes ses props
  ✅ Tester toutes les variantes (default, featured, compact...)
  ✅ Documentation vivante pour les designers et clients
  ✅ Développer sans avoir besoin d'un Drupal fonctionnel
```

---

## Installation

```bash
# Storybook pour Drupal SDC
cd web/themes/custom/mon_theme
npm install --save-dev @storybook/html @storybook/addon-essentials

# Story writer pour SDC
npm install --save-dev @lullabot/storybook-drupal-addon

# Initialiser Storybook
npx storybook@latest init --type html
```

```json
// package.json — scripts
{
  "scripts": {
    "storybook": "storybook dev -p 6006",
    "build-storybook": "storybook build -o storybook-static/"
  }
}
```

---

## Structure d'une Story SDC

```
web/themes/custom/mon_theme/components/
└── card/
    ├── card.component.yml
    ├── card.twig
    ├── card.css
    └── card.stories.js     ← Story Storybook
```

---

## `card.stories.js` — Story Complète

```javascript
// card.stories.js
import card from './card.twig';

// Métadonnées de la story
export default {
  title: 'Composants/Card',
  component: card,
  argTypes: {
    title: {
      control: 'text',
      description: 'Titre principal de la carte',
    },
    url: {
      control: 'text',
    },
    summary: {
      control: 'text',
    },
    variant: {
      control: 'select',
      options: ['default', 'featured', 'compact'],
    },
  },
};

// Template de base — utilisé par toutes les stories
const Template = (args) => card(args);

// ── Stories ───────────────────────────────────────────────────────────────

// Story par défaut
export const Default = Template.bind({});
Default.args = {
  title: 'Introduction à Drupal 11',
  url: '/articles/introduction-drupal-11',
  summary: 'Découvrez les nouveautés de Drupal 11 et comment migrer depuis Drupal 10.',
  date: '15 mai 2026',
  image_url: 'https://picsum.photos/400/250',
  image_alt: 'Illustration Drupal 11',
  variant: 'default',
};

// Story sans image
export const WithoutImage = Template.bind({});
WithoutImage.args = {
  title: 'Article sans image',
  url: '/articles/sans-image',
  summary: 'Cet article ne comporte pas d\'image principale.',
  variant: 'default',
};

// Story variante featured
export const Featured = Template.bind({});
Featured.args = {
  ...Default.args,
  title: 'Article mis en avant',
  variant: 'featured',
};

// Story compacte
export const Compact = Template.bind({});
Compact.args = {
  title: 'Article compact',
  url: '/articles/compact',
  variant: 'compact',
};

// Story avec titre long
export const LongTitle = Template.bind({});
LongTitle.args = {
  title: 'Titre très long qui peut potentiellement déborder sur plusieurs lignes selon la largeur du conteneur',
  url: '/articles/long-title',
  summary: 'Test de débordement.',
  variant: 'default',
};
```

---

## Configuration Storybook pour Twig

```javascript
// .storybook/main.js
/** @type { import('@storybook/html').StorybookConfig } */
const config = {
  stories: ['../components/**/*.stories.js'],
  addons: [
    '@storybook/addon-essentials',
    '@storybook/addon-a11y',    // Audit accessibilité
  ],
  framework: {
    name: '@storybook/html',
    options: {},
  },
  // Twig est compilé automatiquement avec le bon plugin
};
export default config;
```

```javascript
// .storybook/preview.js
/** @type { import('@storybook/html').Preview } */
const preview = {
  parameters: {
    layout: 'centered',
    backgrounds: {
      default: 'light',
      values: [
        { name: 'light', value: '#ffffff' },
        { name: 'dark', value: '#1a1a2e' },
        { name: 'grey', value: '#f5f5f5' },
      ],
    },
  },
};
export default preview;
```

---

## Lancer Storybook dans Docker

```yaml
# docker-compose.yml — service Storybook
storybook:
  image: node:22-alpine
  working_dir: /app/web/themes/custom/mon_theme
  volumes:
    - .:/app
  command: npm run storybook
  ports:
    - "6006:6006"
  environment:
    - NODE_ENV=development
```

```bash
# Démarrer Storybook
docker compose up storybook -d
# → http://localhost:6006

# Build static pour déploiement
docker compose run --rm storybook npm run build-storybook
```

---

## Workflow avec SDC + Storybook

```
1. Designer crée les maquettes (Figma)
         ↓
2. Dev crée le composant SDC (twig + css + component.yml)
         ↓
3. Dev écrit les stories Storybook (toutes les variantes)
         ↓
4. Designer valide dans Storybook (sans Drupal)
         ↓
5. Dev intègre le composant dans les templates Drupal
         ↓
6. Recette visuelle finale dans Drupal
```
