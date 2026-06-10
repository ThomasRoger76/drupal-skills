---
name: drupal-paragraphs
description: Use when building modular content with the Drupal Paragraphs module - creating paragraph types and nested paragraphs, configuring entity reference revisions fields (cardinality, widgets classic vs experimental/paragraphs_ee), implementing ParagraphsBehaviorInterface behaviors (layout options, styles per paragraph), theming paragraphs (paragraph.html.twig suggestions, preprocess), avoiding N+1 rendering issues, translating paragraphs (asymmetric vs symmetric), migrating content from/to Paragraphs (Layout Builder, plain fields), choosing Paragraphs vs Layout Builder vs SDC, cloning paragraphs with entity_clone caveats, or debugging orphaned paragraph revisions in Drupal 8-11+
---

# Drupal Paragraphs

## Installation et configuration

```bash
docker compose exec php composer require drupal/paragraphs
docker compose exec php drush en paragraphs -y
docker compose exec php drush cr
```

## Création d'un type de paragraphe

1. Structure → Types de paragraphes → Ajouter
2. Nommer avec un machine name descriptif (`text_image`, `hero_banner`, `card_grid`)
3. Ajouter les champs nécessaires
4. Exporter la config : `drush cex`

## Ajouter un champ Paragraphs sur un Content Type

1. Structure → Types de contenu → [Type] → Gérer les champs
2. Ajouter champ → Référence → Autres → Paragraphe
3. Configurer les types de paragraphes autorisés
4. Form display : widget "Paragraphs Classic" ou "Paragraphs"
5. Exporter la config

## Patterns de theming

```twig
{# Template : paragraph--text-image.html.twig #}
<div{{ attributes.addClass('paragraph', 'paragraph--type--text-image') }}>
  <div class="paragraph__content">
    {{ content.field_text }}
    {{ content.field_image }}
  </div>
</div>
```

Nommage templates : `paragraph--{type-machine-name}.html.twig`

## Render array en PHP

```php
// Rendu d'un paragraphe
$view_builder = \Drupal::entityTypeManager()->getViewBuilder('paragraph');
$build = $view_builder->view($paragraph, 'full');
```

## Paragraphs Behaviors (D9+) — comportement réutilisable sans champ

Plugin `ParagraphsBehaviorInterface` : ajoute des options (layout, classes CSS, ancre…) à un type
de paragraphe sans créer de champ. Préférer ça à la multiplication de champs de présentation.
```php
// src/Plugin/paragraphs/Behavior/BackgroundBehavior.php
#[ParagraphsBehavior(
  id: 'background_behavior',
  label: new TranslatableMarkup('Background'),
  description: new TranslatableMarkup('Choix de fond'),
)]
class BackgroundBehavior extends ParagraphsBehaviorBase { /* buildBehaviorForm / view */ }
```

## Performance — éviter le N+1

Charger des nœuds avec beaucoup de paragraphes référencés peut générer une requête par paragraphe.
- Le rendu via le view builder gère le cache, mais sur du chargement programmatique massif, précharger :
  `\Drupal::entityTypeManager()->getStorage('paragraph')->loadMultiple($ids)`.
- Surveiller avec la barre Webprofiler / `EXPLAIN` (voir `drupal-performance`).

## Migration depuis un autre système

> `migrate_plus` et `migrate_tools` sont **contrib** (`composer require drupal/migrate_plus drupal/migrate_tools`).
> `entity_reference_revisions` (dépendance de Paragraphs) fournit le destination plugin.

```yaml
# migrate_plus.migration.paragraphs_text.yml
id: paragraphs_text
label: 'Migrate text paragraphs'
source:
  plugin: your_source
process:
  type:
    plugin: default_value
    default_value: text
  field_text/value: body
  field_text/format:
    plugin: default_value
    default_value: full_html
destination:
  plugin: 'entity_reference_revisions:paragraph'
```

## Bonnes pratiques

- Un type de paragraphe = une responsabilité (ne pas créer de "mega paragraphe")
- Limiter le nombre de types exposés par Content Type pour l'éditorial
- Toujours ajouter `paragraph.paragraphs_type.{name}` dans `config/sync`
- Éviter les paragraphes imbriqués > 2 niveaux
- Utiliser `field_paragraph_* naming convention` pour les champs de paragraphe

## Modules complémentaires courants

- `paragraphs_edit` — meilleure UX d'édition
- `bootstrap_paragraphs` — templates Bootstrap ready
- `paragraphs_previewer` — prévisualisation in-place
- `paragraphs_asymmetric_translation_widgets` — traductions asymétriques
