---
name: drupal-seo — metatag
description: Configuration complète du module Metatag pour Drupal - meta title, description, Open Graph, Twitter Cards, canonical, robots, hreflang, et tokens.
---

# Metatag — Référence Complète

## Installation

```bash
composer require drupal/metatag
drush en metatag metatag_open_graph metatag_twitter_cards metatag_hreflang -y
# Modules disponibles : metatag_open_graph, metatag_twitter_cards, metatag_hreflang,
#   metatag_dc, metatag_favicons, metatag_views, metatag_verification
```

---

## Configuration Globale

```
/admin/config/search/metatag/global

Configurer les tags par défaut pour TOUT le site :
├── title : [current-page:title] | [site:name]
├── description : [current-page:summary]
├── robots : index, follow
└── canonical_url : [current-page:url]
```

---

## Configuration par Type de Contenu

```
/admin/config/search/metatag

→ Cliquer "Add default meta tags" → "Content: Article"
→ Configurer les tokens pour ce type de contenu

Tokens essentiels par type :
```

```yaml
# Article / Blog post
title: "[node:title] | [site:name]"
description: "[node:summary]"           # Body summary si défini
# OU
description: "[node:body:summary]"      # Résumé automatique du body

# Open Graph
# ⚠️ Le token image dépend du type de champ :
#   - Champ Image (File direct)  : [node:field_image:entity:url]
#   - Champ Media (image)        : [node:field_image:entity:field_media_image:entity:url]
# Le token …:file:url échoue silencieusement avec un champ Media (cf. lessons.md).
og_title: "[node:title]"
og_description: "[node:summary]"
og_image: "[node:field_image:entity:field_media_image:entity:url]"   # cas Media
og_image_width: "[node:field_image:entity:field_media_image:width]"
og_image_height: "[node:field_image:entity:field_media_image:height]"
og_type: article
og_url: "[node:url]"

# Twitter Cards
twitter_card: summary_large_image
twitter_title: "[node:title]"
twitter_description: "[node:summary]"
twitter_image: "[node:field_image:entity:field_media_image:entity:url]"

# Advanced
canonical_url: "[node:url]"
```

---

## Tokens Metatag — Référence

```bash
# Voir tous les tokens disponibles
drush php:eval "
\$token = \Drupal::service('token');
\$info = \$token->getInfo();
foreach (array_keys(\$info['types']) as \$type) {
  echo \$type . PHP_EOL;
}
"

# Tokens courants pour les nœuds :
# [node:title]                → Titre du nœud
# [node:body]                 → Corps complet
# [node:body:summary]         → Résumé automatique (300 chars)
# [node:summary]              → Champ résumé si défini
# [node:url]                  → URL absolue du nœud
# [node:created]              → Date de création
# [node:changed]              → Date de modification
# [node:author:name]          → Nom de l'auteur
# [node:field_image:entity:file:url]   → URL image (champ Image / File direct)
# [node:field_image:entity:field_media_image:entity:url] → URL image (champ Media)
# [node:field_tags:0:entity:name]      → Premier tag
# [site:name]                 → Nom du site
# [site:url]                  → URL du site
# [current-page:title]        → Titre de la page courante
# [current-page:url]          → URL de la page courante
```

---

## Remplacements par Code — `hook_metatags_attachments_alter()`

```php
<?php

/**
 * Implements hook_metatags_attachments_alter().
 *
 * Modifier les meta tags avant qu'ils soient attachés à la page.
 */
function mon_module_metatags_attachments_alter(array &$metatag_attachments): void {
  // Accéder au nœud courant
  $node = \Drupal::routeMatch()->getParameter('node');
  if (!$node instanceof \Drupal\node\NodeInterface) {
    return;
  }

  foreach ($metatag_attachments['#attached']['html_head'] as &$tag) {
    if (!isset($tag[1])) { continue; }

    // Modifier la meta description pour les articles trop longs
    if ($tag[1] === 'description') {
      $description = $tag[0]['#attributes']['content'] ?? '';
      if (strlen($description) > 160) {
        $tag[0]['#attributes']['content'] = substr($description, 0, 157) . '...';
      }
    }

    // Ajouter article:author pour Open Graph sur les articles
    if ($tag[1] === 'og:type' && $node->bundle() === 'article') {
      $metatag_attachments['#attached']['html_head'][] = [
        [
          '#tag' => 'meta',
          '#attributes' => [
            'property' => 'article:author',
            'content' => $node->getOwner()->getDisplayName(),
          ],
        ],
        'og_article_author',
      ];
    }
  }
}
```

---

## noindex — Pages à Exclure de l'Indexation

```
/admin/config/search/metatag → "Taxonomy term" ou "User" → robots: noindex, follow
```

```php
// Via EventSubscriber — noindex sur les pages de résultats de recherche
function mon_module_metatags_attachments_alter(array &$metatag_attachments): void {
  $route = \Drupal::routeMatch()->getRouteName();

  // Exclure les pages de recherche de l'indexation
  if ($route === 'view.recherche.page_1') {
    foreach ($metatag_attachments['#attached']['html_head'] as &$tag) {
      if (isset($tag[1]) && $tag[1] === 'robots') {
        $tag[0]['#attributes']['content'] = 'noindex, follow';
      }
    }
  }
}
```

---

## hreflang — SEO Multilingue

```bash
drush en metatag_hreflang -y
```

```
/admin/config/search/metatag → "Global" → hreflang section

Configuration automatique :
  - Utilise les traductions disponibles du nœud
  - Génère un <link rel="alternate" hreflang="fr" href="..."> par langue
  - Ajoute x-default pointing vers la langue par défaut
```

---

## Contenu Dupliqué — Canonical et Pagination

```php
// Canonical sur les pages paginées — éviter le duplicate content
function mon_module_preprocess_html(array &$variables): void {
  $request = \Drupal::request();
  $page = (int) $request->query->get('page', 0);

  if ($page > 0) {
    // Page 2+ → canonical vers la page 1 (sans paramètre page)
    $canonical_url = \Drupal\Core\Url::fromRoute(
      '<current>',
      [],
      ['query' => [], 'absolute' => TRUE]
    )->toString();

    $variables['#attached']['html_head'][] = [
      [
        '#tag' => 'link',
        '#attributes' => ['rel' => 'canonical', 'href' => $canonical_url],
      ],
      'canonical_paginated',
    ];
  }
}
```

---

## Déboguer les Meta Tags

```bash
# Vérifier les meta tags générés pour un nœud
drush php:eval "
\$node = \Drupal::entityTypeManager()->getStorage('node')->load(42);
\$metatag_manager = \Drupal::service('metatag.manager');
\$tags = \$metatag_manager->tagsFromEntityWithDefaults(\$node);
foreach (\$tags as \$id => \$tag) {
  echo \$id . ': ' . \$tag . PHP_EOL;
}
"

# Vérifier avec curl (sans JS)
curl -s https://mon-site.com/node/42 | grep -E '<meta|<title|<link rel="canonical"'

# Tester l'Open Graph
# https://developers.facebook.com/tools/debug/ (Facebook Sharing Debugger)
# https://cards-dev.twitter.com/validator (Twitter Card Validator)
# https://search.google.com/test/rich-results (Google Rich Results Test)
```
