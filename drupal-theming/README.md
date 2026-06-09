# Claude Code Skill — drupal-theming

> Claude Code skill — Theming Drupal : Twig, preprocess, libraries.yml, responsive images, templates (D8-D11+)

## Installation

```bash
git clone https://github.com/ThomasRoger76/drupal-theming ~/.claude/skills/drupal-theming
```

Redémarrer Claude Code pour que le skill soit détecté.

## Mise à jour

```bash
cd ~/.claude/skills/drupal-theming && git pull
```

## Fichiers inclus

- SKILL.md — point d'entrée, Quick Decision Table, anti-patterns, versioning
- theme-anatomy.md — `.info.yml`, régions, base themes, SDC
- libraries-assets.md — `.libraries.yml`, override/extend, behaviors, drupalSettings
- twig-templates.md — filtres, `attributes`, render arrays, héritage Twig
- theme-suggestions.md — Twig debug, `hook_theme_suggestions_HOOK_alter()`
- preprocess.md — preprocess hooks, cache tags, `hook_*_alter`
- responsive-images.md — `breakpoints.yml`, image styles, WebP, focal_point
- twig3-accessibility.md — Twig 3 (arrow functions, apply), ARIA/WCAG
- accessibility.md — WCAG 2.1 AA, RGAA, tests automatisés
- css-moderne.md — Container Queries, dark mode, logical properties, cascade layers
- bem-storybook.md — BEM + Storybook/SDC
- bootstrap5.md — sous-thème Bootstrap 5
- build-pipeline.md — Vite / Webpack / Gulp, service Node Docker
- email-templates.md — templates email Symfony Mailer
- agents/theme-generator.md — générateur de thème D11
- lessons.md — bugs rencontrés en usage réel
- CHANGELOG.md — historique des versions

## Suite des skills Drupal

| Skill | Couverture |
|-------|-----------|
| [drupal-core](https://github.com/ThomasRoger76/drupal-core) | Architecture, DI, Entity API |
| [drupal-theming](https://github.com/ThomasRoger76/drupal-theming) | Twig, preprocess, libraries |
| [drupal-config](https://github.com/ThomasRoger76/drupal-config) | Config Management, UUID |
| [drupal-testing](https://github.com/ThomasRoger76/drupal-testing) | PHPUnit, CI/CD |
| [drupal-security](https://github.com/ThomasRoger76/drupal-security) | XSS, CSRF, SQL injection |
| [drupal-docker](https://github.com/ThomasRoger76/drupal-docker) | Docker, DDEV, Makefile |
| [drupal-obsidian](https://github.com/ThomasRoger76/drupal-obsidian) | Architecture → Obsidian |

## Licence

MIT
