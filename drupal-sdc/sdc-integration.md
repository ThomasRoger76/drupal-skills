---
name: drupal-sdc — intégration
description: Utiliser les SDC Drupal dans les templates Twig, les modules, Layout Builder, et composer des composants imbriqués.
---

# SDC Intégration — Utilisation dans Drupal

## Utiliser un Composant dans un Template Twig

```twig
{# Dans n'importe quel template Drupal #}

{# Syntaxe 1 : include avec props explicites #}
{% include 'mon_theme:card' with {
  title: node.title.value,
  url: node.url,
  summary: node.body.summary ?: node.body.value|striptags|slice(0, 200),
  image_url: node.field_image.entity.field_media_image.entity.uri.value|file_url,
  image_alt: node.field_image.entity.field_media_image.alt,
  variant: 'featured',
} only %}

{# Syntaxe 2 : via la fonction component() #}
{{ component('mon_theme:card', {
  title: 'Mon Titre',
  url: path('entity.node.canonical', {node: node.id}),
  summary: 'Description...',
}) }}

{# Syntaxe 3 : avec des slots #}
{% embed 'mon_theme:card' with {
  title: node.title.value,
  url: node.url,
} %}
  {% block badge %}
    <span class="badge badge--new">{{ 'Nouveau'|t }}</span>
  {% endblock %}
  {% block footer %}
    <a href="{{ node.url }}" class="btn">{{ 'Lire la suite'|t }}</a>
  {% endblock %}
{% endembed %}
```

---

## SDC dans un Module Custom

```
web/modules/custom/mon_module/components/
└── alert/
    ├── alert.component.yml
    ├── alert.twig
    └── alert.css
```

```yaml
# alert.component.yml
$schema: 'https://git.drupalcode.org/project/drupal/-/raw/HEAD/core/assets/schemas/v1/metadata.schema.json'
name: Alert
group: Mon Module
props:
  type: object
  required: [message]
  properties:
    message:
      type: string
    type:
      type: string
      enum: [info, warning, error, success]
      default: info
slots:
  actions:
    title: Actions
```

```twig
{# alert.twig #}
<div class="alert alert--{{ type|default('info') }}" role="alert">
  <p class="alert__message">{{ message }}</p>
  {% if actions %}
    <div class="alert__actions">{{ actions }}</div>
  {% endif %}
</div>
```

```twig
{# Utilisation dans un template #}
{% include 'mon_module:alert' with {
  message: 'Votre profil a été mis à jour.',
  type: 'success',
} only %}
```

---

## SDC dans un Render Array (PHP)

```php
// Dans un Controller ou Block::build()
return [
  '#type' => 'component',
  '#component' => 'mon_theme:card',
  '#props' => [
    'title' => $node->getTitle(),
    'url' => $node->toUrl()->toString(),
    'summary' => $node->get('body')->summary,
  ],
  '#slots' => [
    'footer' => [
      '#markup' => '<a href="' . $node->toUrl()->toString() . '">Lire la suite</a>',
    ],
  ],
];
```

---

## Composants Imbriqués

```yaml
# section.component.yml — composant qui contient des cards
name: Section
props:
  type: object
  required: [title]
  properties:
    title:
      type: string
slots:
  items:
    title: Éléments de la section
    description: Composants card à afficher.
```

```twig
{# section.twig #}
<section class="section">
  <h2 class="section__title">{{ title }}</h2>
  <div class="section__items">
    {{ items }}
  </div>
</section>
```

```twig
{# Dans page.html.twig — imbriquer section > cards #}
{% embed 'mon_theme:section' with { title: 'Articles récents' } %}
  {% block items %}
    {% for node in articles %}
      {% include 'mon_theme:card' with {
        title: node.title.value,
        url: node.url,
      } only %}
    {% endfor %}
  {% endblock %}
{% endembed %}
```

---

## SDC avec Preprocess

```php
// Dans mon_theme.theme — préparer les props avant le rendu SDC
function mon_theme_preprocess_node__article(&$variables): void {
  $node = $variables['node'];

  // Préparer les props pour le composant SDC
  $variables['card_props'] = [
    'title' => $node->getTitle(),
    'url' => $node->toUrl()->toString(),
    'summary' => $node->get('body')->summary ?: '',
    'variant' => $node->get('field_featured')->value ? 'featured' : 'default',
  ];

  // Pour l'image
  if (!$node->get('field_image')->isEmpty()) {
    $media = $node->get('field_image')->entity;
    if ($media) {
      $file = $media->get('field_media_image')->entity;
      if ($file) {
        $variables['card_props']['image_url'] = \Drupal::service('file_url_generator')
          ->generateAbsoluteString($file->getFileUri());
        $variables['card_props']['image_alt'] = $media->get('field_media_image')->alt;
      }
    }
  }
}
```

```twig
{# node--article--teaser.html.twig #}
{% include 'mon_theme:card' with card_props only %}
```

---

## Commandes de Debug

```bash
# Lister les composants disponibles avec leur ID
drush php:eval "
\$r = \Drupal::service('sdc.component_registry');
foreach (\$r->getAllComponents() as \$id => \$c) {
  echo \$id . ' (' . \$c->getPath() . ')' . PHP_EOL;
}
"

# Vider le cache SDC
drush cr

# Activer les erreurs de validation SDC
# services.yml → parameters.sdc.debug: true
```
