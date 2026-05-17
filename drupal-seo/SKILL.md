---
name: drupal-seo
description: Use when configuring SEO on Drupal sites with the Metatag module (meta title, description, Open Graph for Facebook/LinkedIn, Twitter Cards, canonical URL, robots meta), generating XML sitemaps with drupal/simple_sitemap (per-entity, per-language, image sitemap, priority/changefreq), configuring URL aliases with drupal/pathauto (patterns with tokens, bulk generate, redirect on alias change), implementing 301/302 redirects with drupal/redirect, adding Schema.org JSON-LD structured data (Article, BreadcrumbList, Organization, Product), configuring hreflang for multilingual SEO, preventing duplicate content (canonical, noindex), setting up Google Search Console verification, or auditing Drupal SEO health in Drupal 8-11+
---

# Drupal SEO — Référence Complète

## Overview

Référentiel complet du SEO Drupal 8-11+ : Metatag (meta tags, Open Graph, Twitter Cards, Schema.org), Simple Sitemap (XML sitemap), Pathauto (URL aliases), Redirect (301/302), données structurées JSON-LD, hreflang multilingue, et checklist SEO production.

## 🎯 La Règle Fondamentale

> **SEO en couches.** Metatag gère le `<head>`, Pathauto gère les URLs, Simple Sitemap gère la découverte, Schema.org gère la compréhension. Ces quatre couches sont indépendantes et complémentaires — toutes sont nécessaires pour un SEO complet.

---

## Quick Decision Table

| Besoin | Module / Outil | Référence |
|--------|---------------|-----------|
| Meta title et description | `drupal/metatag` | [metatag.md](metatag.md) |
| Open Graph (partage Facebook, LinkedIn) | Metatag → OG groupe | [metatag.md](metatag.md) |
| Twitter Cards (partage Twitter/X) | Metatag → Twitter Cards groupe | [metatag.md](metatag.md) |
| URL canonique (`<link rel="canonical">`) | Metatag → Advanced → Canonical URL | [metatag.md](metatag.md) |
| noindex sur certaines pages | Metatag → robots meta + config | [metatag.md](metatag.md) |
| Google Search Console vérification | Metatag → Global → Verification | [metatag.md](metatag.md) |
| Lier les champs Drupal aux meta tags | Metatag tokens (`[node:title]`, `[node:summary]`) | [metatag.md](metatag.md) |
| Sitemap XML (toutes les entités) | `drupal/simple_sitemap` | [sitemap.md](sitemap.md) |
| Sitemap image (Google Images) | Simple Sitemap → Image sitemap | [sitemap.md](sitemap.md) |
| Sitemap par langue (hreflang dans sitemap) | Simple Sitemap → multilingual | [sitemap.md](sitemap.md) |
| Exclure certaines pages du sitemap | Simple Sitemap → entity exclusion | [sitemap.md](sitemap.md) |
| Priority et changefreq par type de contenu | Simple Sitemap → entity type settings | [sitemap.md](sitemap.md) |
| URL propre `/articles/mon-titre` | `drupal/pathauto` + patterns | [pathauto.md](pathauto.md) |
| Redirection 301 automatique si alias change | Pathauto → Redirect on update | [pathauto.md](pathauto.md) |
| Génération en masse des alias | `drush pathauto:aliases-generate` | [pathauto.md](pathauto.md) |
| Tokens Pathauto custom | `hook_token_info()` + `hook_tokens()` | [pathauto.md](pathauto.md) |
| Redirection 301 de l'ancien vers le nouveau chemin | `drupal/redirect` | [redirects.md](redirects.md) |
| Import de redirections en masse (CSV) | Module `redirect_import` | [redirects.md](redirects.md) |
| Schema.org Article (articles de blog) | JSON-LD via Metatag ou hook | [structured-data.md](structured-data.md) |
| Schema.org BreadcrumbList | JSON-LD dans hook_page_attachments | [structured-data.md](structured-data.md) |
| Schema.org Organization (footer) | JSON-LD global via preprocess_html | [structured-data.md](structured-data.md) |
| Schema.org Product (e-commerce) | `drupal/metatag` ou JSON-LD custom | [structured-data.md](structured-data.md) |
| hreflang pour le SEO multilingue | Metatag hreflang group | [metatag.md](metatag.md) |
| Prévenir le contenu dupliqué (pagination) | Canonical + rel="next/prev" | [metatag.md](metatag.md) |
| Audit SEO complet | Agent `/drupal-seo-audit` | [agents/seo-audit.md](agents/seo-audit.md) |
| Vérifier les meta tags en développement | Module `metatag_extended_perms` + DevTools | [metatag.md](metatag.md) |
| **Analytics : Google Analytics (GA4)** | `drupal/google_analytics` ou `drupal/google_tag` (GTM) | [analytics.md](analytics.md) |
| **Analytics : Matomo (auto-hébergé)** | `drupal/matomo` — sans tiers, RGPD natif | [analytics.md](analytics.md) |
| **Cookie consent RGPD** | `drupal/tarte_au_citron` (FR) ou `drupal/cookieyes` | [analytics.md](analytics.md) |
| Bloquer analytics avant consentement | Tarte au Citron → services Google/Matomo gérés | [analytics.md](analytics.md) |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| Meta title identique sur toutes les pages | Tokens par type de contenu (`[node:title] | [site:name]`) | Duplicate title penalty |
| Meta description vide | Fallback sur `[node:summary]` ou `[node:body:summary]` | CTR réduit dans les SERPs |
| `<meta name="robots" content="noindex">` oublié en production | Vérifier `settings.php` + Metatag config prod | Site non indexé |
| URL non configurées (node/42) | Pathauto patterns + `drush pathauto:aliases-generate` | URLs laides, mal indexées |
| Pas de redirection 301 après changement d'alias | `redirect_on_update: true` dans Pathauto | 404 pour Google + perte de jus de lien |
| Sitemap sans images | Simple Sitemap + Image sitemap activé | Images non indexées dans Google Images |
| Open Graph image sans dimensions (og:image:width/height) | Dimensions dans Metatag token | Rendu social incorrect |
| Schema.org JSON-LD non validé | Tester sur schema.org/validator | Rich snippets non activés |
| Canonical absent sur les pages paginées | Canonical self-referencing sur chaque page | Duplicate content pénalité |
| `drush pathauto:aliases-generate` jamais lancé | Planifier en cron ou après import | Centaines de node/42 non résolus |

## Évolution par Version Majeure

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| Metatag module | contrib | contrib | contrib | contrib |
| Simple Sitemap | contrib | contrib | contrib | contrib |
| Pathauto | contrib | contrib | contrib | contrib |
| Redirect | contrib | contrib | contrib | contrib |
| Token module (base) | contrib | contrib | contrib | contrib |
| JSON-LD via hook | ✅ | ✅ | ✅ | ✅ |
| hreflang automatique | contrib | contrib | contrib | contrib |
| Core Web Vitals (LCP, CLS, INP) | ❌ | ❌ | ⚠️ émergent | ✅ key signal |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Problèmes SEO découverts en projet réel (indexation, rich snippets, redirects perdus).
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions du skill.

**Workflow :** problème SEO découvert → corriger dans le projet → ajouter entrée dans lessons.md → incrémenter CHANGELOG.

## See Also

- `drupal-multilingual` — hreflang détaillé, URL par langue, config translation
- `drupal-performance` — Core Web Vitals, LCP, CLS, INP (signaux SEO)
- `drupal-theming` — meta tags dans preprocess_html, preconnect, preload
- `drupal-content-modeling` — Schema.org par type de contenu, champs SEO
- `drupal-config` — Export/import config Metatag, Pathauto patterns
- `drupal-migration` — Préserver les alias URL et redirections lors des migrations
- `drupal-multilingual` — hreflang détaillé, URL par langue

---

## Complémentarité avec les Skills SEO Génériques

> Ce skill est **différent** des skills SEO génériques (`seo`, `seo-audit`, `seo-technical`, `seo-schema`).

| Skill | Rôle |
|-------|------|
| `seo`, `seo-audit` | **Analyser** le SEO d'un site (audit, scores, recommandations) |
| `seo-technical` | **Diagnostiquer** les problèmes techniques (crawlabilité, vitesse) |
| `seo-schema` | **Valider** les données structurées Schema.org |
| **`drupal-seo`** | **Implémenter** le SEO dans Drupal (Metatag, Pathauto, Simple Sitemap, JSON-LD via hooks) |

**Workflow :** Le skill `seo-audit` identifie le problème ("meta descriptions manquantes") → `drupal-seo` explique comment le résoudre dans Drupal (configurer Metatag avec les bons tokens).
