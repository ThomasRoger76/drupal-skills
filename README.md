# drupal-skills

> 24 Claude Code skills Drupal — installation en une commande via `npx skills`

## Installation

```bash
# Tout installer (24 skills)
npx skills add ThomasRoger76/drupal-skills --all

# Sélection interactive
npx skills add ThomasRoger76/drupal-skills

# Un skill précis
npx skills add ThomasRoger76/drupal-skills --skill drupal-core

# Plusieurs skills
npx skills add ThomasRoger76/drupal-skills --skill drupal-core --skill drupal-theming

# Lister les skills disponibles sans installer
npx skills add ThomasRoger76/drupal-skills --list
```

## Mise à jour

```bash
npx skills update
```

## Skills disponibles (24)

| Skill | Description |
|-------|-------------|
| `drupal-api` | JSON:API, REST, GraphQL, Next.js Drupal |
| `drupal-composer` | Composer, patches, dépôts privés, sécurité |
| `drupal-config` | Config Management, UUID, config_split, Recipes |
| `drupal-content-modeling` | Content Types, Paragraphs, Taxonomie, Layout Builder |
| `drupal-core` | Modules custom, hooks, plugins, services, Forms API |
| `drupal-cron-queue` | Cron, Queue Workers, Batch API |
| `drupal-deployment` | CI/CD, Platform.sh, Acquia, Pantheon, zero-downtime |
| `drupal-docker` | Docker, DDEV, Kubernetes, Makefile, Taskfile |
| `drupal-email` | Symfony Mailer, SMTP, templates, deliverability |
| `drupal-gin` | Thème admin Gin, sous-thème, navigation |
| `drupal-media` | Media Library, focal point, oEmbed, accès privé |
| `drupal-migration` | Migrate API, D7→D11, CSV/XML/JSON, Rector |
| `drupal-multilingual` | Multilingue, négociation langue, TMGMT, RTL |
| `drupal-obsidian` | Mapping Drupal → Obsidian vault + graphify |
| `drupal-performance` | Cache, Redis, Varnish, CDN, profiling, opcache |
| `drupal-sdc` | Single Directory Components, Storybook, Layout Builder |
| `drupal-search` | Search API, Solr, Elasticsearch, Facets, Views |
| `drupal-security` | XSS, CSRF, SQL injection, OAuth, audit sécurité |
| `drupal-seo` | Metatag, Pathauto, sitemap, Schema.org, hreflang |
| `drupal-testing` | PHPUnit, DTT, CI/CD, PHPStan, Rector |
| `drupal-theming` | Twig, preprocess, Bootstrap5, SDC, BEM, accessibilité |
| `drupal-token` | Token API, tokens custom, intégration modules |
| `drupal-views` | Views, handlers custom, REST export, cache, templates |
| `drupal-webform` | Webform, éléments, handlers, soumissions, accès |

## Agents supportés

Ces skills fonctionnent avec tous les agents compatibles `npx skills` : Claude Code, Cursor, Copilot, Gemini CLI, et plus.

## Architecture

Chaque skill est maintenu dans son propre dépôt GitHub (`ThomasRoger76/drupal-*`) et synchronisé automatiquement dans ce monorepo chaque nuit via GitHub Actions.

Pour installer un skill individuellement :
```bash
git clone https://github.com/ThomasRoger76/drupal-core ~/.claude/skills/drupal-core
```
