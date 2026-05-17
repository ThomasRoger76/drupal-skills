---
name: drupal-seo — structured data Schema.org
description: Implémenter des données structurées JSON-LD Schema.org dans Drupal pour les rich snippets Google - Article, BreadcrumbList, Organization, Product, FAQPage.
---

# Données Structurées JSON-LD — Référence Complète

## Pourquoi JSON-LD pour Drupal

Google recommande JSON-LD (inline `<script>`) plutôt que les microdata ou RDFa. Les avantages pour Drupal :
- Séparation claire du HTML
- Facile à générer depuis PHP
- Pas de modification des templates Twig
- Validation indépendante

---

## Via Metatag (Module)

```bash
# Module metatag inclut un groupe Schema.org
composer require drupal/metatag

# Groupes disponibles :
# metatag_schema_article, metatag_schema_organization, metatag_schema_product
```

```
/admin/config/search/metatag → "Content: Article"
→ Ajouter des tags du groupe "Schema.org"
  ├── Schema.org: Article - @type: Article
  ├── Schema.org: Article - name: [node:title]
  ├── Schema.org: Article - datePublished: [node:created:html_datetime]
  ├── Schema.org: Article - dateModified: [node:changed:html_datetime]
  └── Schema.org: Article - author: [node:author:name]
```

---

## Via hook_page_attachments (JSON-LD Manuel)

```php
<?php
/**
 * Implements hook_page_attachments().
 *
 * Ajouter du JSON-LD Schema.org sur les pages pertinentes.
 */
function mon_module_page_attachments(array &$page): void {
  $node = \Drupal::routeMatch()->getParameter('node');

  if (!$node instanceof \Drupal\node\NodeInterface) {
    return;
  }

  $schema = NULL;

  // Article
  if ($node->bundle() === 'article') {
    $schema = [
      '@context' => 'https://schema.org',
      '@type' => 'Article',
      'headline' => $node->getTitle(),
      'datePublished' => date('c', $node->getCreatedTime()),
      'dateModified' => date('c', $node->getChangedTime()),
      'author' => [
        '@type' => 'Person',
        'name' => $node->getOwner()->getDisplayName(),
      ],
      'publisher' => [
        '@type' => 'Organization',
        'name' => \Drupal::config('system.site')->get('name'),
        'logo' => [
          '@type' => 'ImageObject',
          'url' => \Drupal::request()->getSchemeAndHttpHost() . '/themes/custom/mon_theme/logo.svg',
        ],
      ],
      'url' => $node->toUrl('canonical', ['absolute' => TRUE])->toString(),
    ];

    // Image principale si disponible
    if (!$node->get('field_image')->isEmpty()) {
      $media = $node->get('field_image')->entity;
      if ($media) {
        $file = $media->get('field_media_image')->entity;
        if ($file) {
          $file_url = \Drupal::service('file_url_generator')
            ->generateAbsoluteString($file->getFileUri());
          $schema['image'] = [
            '@type' => 'ImageObject',
            'url' => $file_url,
            'width' => $media->get('field_media_image')->width,
            'height' => $media->get('field_media_image')->height,
          ];
        }
      }
    }

    // Description (résumé du body)
    $body = $node->get('body');
    if (!$body->isEmpty() && $body->summary) {
      $schema['description'] = strip_tags($body->summary);
    }
  }

  // FAQ Page
  if ($node->bundle() === 'faq') {
    $questions = [];
    foreach ($node->get('field_questions') as $item) {
      $questions[] = [
        '@type' => 'Question',
        'name' => $item->question,
        'acceptedAnswer' => [
          '@type' => 'Answer',
          'text' => strip_tags($item->answer),
        ],
      ];
    }

    $schema = [
      '@context' => 'https://schema.org',
      '@type' => 'FAQPage',
      'mainEntity' => $questions,
    ];
  }

  if ($schema) {
    $page['#attached']['html_head'][] = [
      [
        '#tag' => 'script',
        '#attributes' => ['type' => 'application/ld+json'],
        '#value' => json_encode($schema, JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT),
      ],
      'schema_org_' . $node->bundle() . '_' . $node->id(),
    ];
  }
}
```

---

## BreadcrumbList — Fil d'Ariane Schema.org

```php
function mon_module_page_attachments(array &$page): void {
  $breadcrumb = \Drupal::service('breadcrumb')
    ->build(\Drupal::routeMatch())
    ->getLinks();

  if (count($breadcrumb) > 1) {
    $items = [];
    foreach ($breadcrumb as $position => $link) {
      $items[] = [
        '@type' => 'ListItem',
        'position' => $position + 1,
        'name' => $link->getText(),
        'item' => $link->getUrl()->setAbsolute()->toString(),
      ];
    }

    $schema = [
      '@context' => 'https://schema.org',
      '@type' => 'BreadcrumbList',
      'itemListElement' => $items,
    ];

    $page['#attached']['html_head'][] = [
      [
        '#tag' => 'script',
        '#attributes' => ['type' => 'application/ld+json'],
        '#value' => json_encode($schema, JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE),
      ],
      'schema_org_breadcrumb',
    ];
  }
}
```

---

## Organization — Données Globales du Site

```php
// Dans hook_preprocess_html — ajouter Organization sur toutes les pages
function mon_module_preprocess_html(array &$variables): void {
  $config = \Drupal::config('system.site');
  $request = \Drupal::request();

  $organization_schema = [
    '@context' => 'https://schema.org',
    '@type' => 'Organization',
    'name' => $config->get('name'),
    'url' => $request->getSchemeAndHttpHost(),
    'logo' => $request->getSchemeAndHttpHost() . '/themes/custom/mon_theme/logo.svg',
    'sameAs' => [
      'https://www.linkedin.com/company/mon-organisation',
      'https://twitter.com/mon_organisation',
    ],
    'contactPoint' => [
      '@type' => 'ContactPoint',
      'contactType' => 'customer service',
      'email' => $config->get('mail'),
    ],
  ];

  $variables['#attached']['html_head'][] = [
    [
      '#tag' => 'script',
      '#attributes' => ['type' => 'application/ld+json'],
      '#value' => json_encode($organization_schema, JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE),
    ],
    'schema_org_organization',
  ];
}
```

---

## Valider les Données Structurées

```bash
# Valider le JSON-LD généré
curl -s https://mon-site.com/node/42 | \
  grep -A 20 'application/ld+json' | \
  python3 -m json.tool

# Validateurs en ligne :
# https://validator.schema.org/ (Google/Schema.org officiel)
# https://search.google.com/test/rich-results (Google Rich Results Test)
# https://developers.facebook.com/tools/debug/ (Open Graph)
```
