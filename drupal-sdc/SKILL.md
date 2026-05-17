---
name: drupal-sdc
description: Use when creating Single Directory Components (SDC) in Drupal 10.3+ and D11 - structuring a component directory with *.component.yml, *.twig template, *.css, and *.js files, defining props (typed schema properties) and slots (Twig regions), using components in other Twig templates with include or the component() function, integrating SDC with Storybook for visual documentation, using SDC in Layout Builder blocks, creating SDC-based theme components, configuring component libraries, debugging SDC prop validation errors, and understanding the difference between SDC props vs Twig variables in Drupal 10.3-11+
---

# Single Directory Components (SDC) — Référence Complète

## Overview

Référentiel complet des Single Directory Components Drupal 10.3+/D11 : structure d'un composant, props typées, slots, intégration Twig, JavaScript, CSS, et Storybook. SDC est l'avenir du theming Drupal — il co-localise template, styles, scripts et définition dans un seul répertoire.

> **Note d'adoption (2024-2025) :** SDC est stable en D11 mais son adoption terrain est encore faible — la majorité des projets en production utilisent encore le theming Twig classique avec Bootstrap5. SDC est recommandé pour les **nouveaux projets D11** ; pour les projets existants D10, évaluer la migration au cas par cas.

## 🎯 La Règle Fondamentale

> **Un composant = un répertoire auto-suffisant.** SDC co-localise TOUT ce qui concerne un composant : template Twig, CSS, JS, props schema, et stories Storybook. Finies les recherches dans 5 répertoires différents.

---

## Quick Decision Table

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Créer un composant SDC | Répertoire `components/mon-composant/` + `.component.yml` | [sdc-setup.md](sdc-setup.md) |
| Définir les props d'un composant | `*.component.yml` → `props:` section | [sdc-setup.md](sdc-setup.md) |
| Définir des slots (régions Twig) | `*.component.yml` → `slots:` section | [sdc-setup.md](sdc-setup.md) |
| Utiliser un composant dans un template | `{% include 'mon_theme:card' with {...} %}` | [sdc-integration.md](sdc-integration.md) |
| Utiliser la fonction component() | `{{ component('mon_theme:card', {title: '...'}) }}` | [sdc-integration.md](sdc-integration.md) |
| Ajouter du CSS scopé à un composant | `mon-composant.css` dans le répertoire | [sdc-setup.md](sdc-setup.md) |
| Ajouter du JS à un composant | `mon-composant.js` avec `Drupal.behaviors` | [sdc-setup.md](sdc-setup.md) |
| Composer des composants imbriqués | `{% include %}` dans le `.twig` d'un autre composant | [sdc-integration.md](sdc-integration.md) |
| Documenter visuellement dans Storybook | `*.stories.js` dans le répertoire | [sdc-storybook.md](sdc-storybook.md) |
| Debug des erreurs de validation de props | `DRUPAL_SDC_DEBUG=1` ou `services.yml` | [sdc-setup.md](sdc-setup.md) |
| SDC dans un module (pas un thème) | `web/modules/custom/mon_module/components/` | [sdc-integration.md](sdc-integration.md) |
| SDC avec Layout Builder — block plugin | Classe Block PHP qui render un composant SDC | [sdc-layout-builder.md](sdc-layout-builder.md) |
| SDC comme région de layout (Section plugin) | `LayoutBase::build()` qui retourne un `#type: component` | [sdc-layout-builder.md](sdc-layout-builder.md) |
| Props SDC configurables dans l'UI Layout Builder | `blockForm()` + `blockSubmit()` + `#type: component` | [sdc-layout-builder.md](sdc-layout-builder.md) |
| Activer SDC en dev (validation stricte) | `services.yml` → `sdc.debug: true` | [sdc-setup.md](sdc-setup.md) |
| Lister tous les composants disponibles | `drush php:eval "print_r(\Drupal::service('sdc.component_registry')->getAllComponents());"` | [sdc-setup.md](sdc-setup.md) |
| **Documenter visuellement les composants** | Storybook + `*.stories.js` co-localisé dans le composant | [sdc-storybook.md](sdc-storybook.md) |
| **Lancer Storybook dans Docker** | Service `node:22-alpine` + `npm run storybook` port 6006 | [sdc-storybook.md](sdc-storybook.md) |
| Story avec plusieurs variantes (default/featured) | `export const Featured = Template.bind({}); Featured.args = {...}` | [sdc-storybook.md](sdc-storybook.md) |
| **JS Drupal.behaviors dans un SDC** | `Drupal.behaviors.monComposant = { attach: (context) => { once(...) } }` | [sdc-setup.md](sdc-setup.md) |
| Props objet imbriqué (image avec alt + url) | `props: { image: { type: object, properties: { url, alt } } }` | [sdc-setup.md](sdc-setup.md) |
| Slot facultatif avec valeur par défaut Twig | `{% if slots.footer is defined %}...{% else %}...{% endif %}` | [sdc-integration.md](sdc-integration.md) |

## Anatomie d'un Composant SDC

```
web/themes/custom/mon_theme/components/
└── card/
    ├── card.component.yml     ← Définition (props, slots, metadata)
    ├── card.twig              ← Template HTML du composant
    ├── card.css               ← Styles scopés au composant
    └── card.js                ← Comportement JS du composant
```

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| Props sans types définis | Toujours typer dans `*.component.yml` | Validation silencieuse, debug difficile |
| JS sans pattern `Drupal.behaviors` | `Drupal.behaviors.monComposant = { attach: ... }` | Double initialisation |
| Importer des CSS externes dans le composant | CSS scopé uniquement, librairies partagées via `libraries.yml` | Conflits de styles |
| `{{ variable\|raw }}` dans un composant | `{{ variable }}` (auto-escape) | XSS |
| Nom de composant avec majuscules ou espaces | Kebab-case : `mon-composant` | Erreurs de découverte |
| Composants imbriqués trop profonds (>3) | Aplatir, utiliser des slots | Performance, lisibilité |

## Évolution par Version Majeure

| Feature | D10.3 | D10.4+ | D11 |
|---------|-------|--------|-----|
| SDC core | ✅ expérimental | ✅ stable | ✅ stable |
| Props validation | ✅ | ✅ stricte | ✅ stricte |
| SDC dans modules | ✅ | ✅ | ✅ |
| Storybook integration | contrib | contrib | contrib |
| Layout Builder + SDC | ✅ | ✅ | ✅ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Erreurs SDC découvertes en projet.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions.

## See Also

- `drupal-theming` — Twig templates, libraries, build pipeline Vite
- `drupal-core` — Plugin Block, hook_theme, render arrays
- `drupal-content-modeling` — Layout Builder avec composants SDC
