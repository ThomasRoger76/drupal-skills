---
name: drupal-layout-builder
description: Use when building visual page layouts with Drupal Layout Builder - enabling Layout Builder on content types (defaults vs per-entity overrides), creating sections (1/2/3 columns, custom layouts via layout plugin + twig), placing blocks (field blocks, custom #[Block] plugins, inline blocks, Views blocks), restricting available blocks/layouts with layout_builder_restrictions, managing permissions (configure any layout, override per entity), exporting layouts in config vs content, migrating from Paragraphs to Layout Builder, styling sections with layout_builder_styles, comparing Layout Builder vs Paragraphs vs SDC vs Experience Builder for a project, or debugging layout rendering issues in Drupal 8-11+
---

# Drupal Layout Builder

## Activation

```bash
docker compose exec php drush en layout_builder layout_discovery -y
docker compose exec php drush cr
```

## Activer Layout Builder sur un Content Type

1. Structure → Types de contenu → [Type] → Gérer l'affichage
2. Onglet "Default" → cocher "Use Layout Builder"
3. Option : "Allow each content item to have its layout customized" pour surcharges par entité
4. Sauvegarder → Configure Layout
5. Exporter : `drush cex`

## Concepts clés

- **Section** : conteneur de layout (1/2/3 colonnes, full width...)
- **Bloc** : composant placé dans une section (champ, bloc custom, vue...)
- **Surcharge** : chaque nœud peut avoir son propre layout si activé
- **Defaults** : layout appliqué à tous les nœuds du type (sans surcharge)

## Créer un bloc custom pour Layout Builder

```php
// src/Plugin/Block/HeroBlock.php — D11 : attribut PHP #[Block] (annotation @Block dépréciée)
namespace Drupal\mymodule\Plugin\Block;

use Drupal\Core\Block\Attribute\Block;
use Drupal\Core\Block\BlockBase;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[Block(
  id: 'hero_block',
  admin_label: new TranslatableMarkup('Hero'),
  category: new TranslatableMarkup('Custom'),
)]
class HeroBlock extends BlockBase {

  public function build(): array {
    return [
      '#theme' => 'hero_block',
      '#title' => $this->configuration['title'] ?? '',
      // Toujours définir le cache sur du contenu dynamique (sinon contenu périmé).
      '#cache' => ['contexts' => ['url.path'], 'tags' => ['node_list']],
    ];
  }

}
```

> **D8-D10** : l'annotation `@Block(...)` reste valide. **D11** : préférer l'attribut `#[Block]`
> (les annotations sont dépréciées). Voir `drupal-core` pour le plugin system complet.

## Sections custom

```php
// Implements hook_layout_alter()
function mymodule_layout_alter(array &$definitions): void {
  // Modifier les définitions de layout existantes
}
```

## Theming

```twig
{# layout--twocol-section.html.twig #}
<div{{ attributes.addClass('layout', 'layout--twocol') }}>
  <div class="layout__region layout__region--first">
    {{ content.first }}
  </div>
  <div class="layout__region layout__region--second">
    {{ content.second }}
  </div>
</div>
```

## Permissions Layout Builder

- `configure any layout` — admin full control
- `configure editable [type] node layout overrides` — éditeurs par type

## Paragraphs vs Layout Builder

| Critère | Paragraphs | Layout Builder |
|---------|-----------|----------------|
| UX éditeur | Formulaire vertical | Drag & drop visuel |
| Contrôle précis des champs | Oui | Limité |
| Surcharge par entité | Non natif | Natif |
| Performance | Bonne | Attention aux surcharges |
| Migration | Facile | Complexe |

## Bonnes pratiques

- Désactiver les surcharges par entité si non nécessaire (complexité + perf)
- Créer des Section Libraries pour les layouts réutilisables
- Exporter les layouts defaults dans la config — ne pas laisser en DB
- `drush cr` après tout changement de template ou plugin de bloc
- Tester les permissions éditeur avant livraison

## Module complémentaire

- `layout_builder_restrictions` — contrôle quels blocs sont disponibles par section
- `layout_builder_styles` — ajouter des classes CSS aux sections/blocs
- `section_library` — sauvegarder et réutiliser des sections

## Experience Builder (XB) — le successeur en approche

**Experience Builder** (`drupal/experience_builder`) est l'initiative stratégique Drupal (Drupal CMS /
Starshot) destinée à remplacer Layout Builder à terme : éditeur visuel en iframe temps réel (React),
composants **SDC** comme briques de base, theming par tokens.

| Critère | Layout Builder | Experience Builder |
|---------|----------------|--------------------|
| Maturité | Stable depuis D8.7 — production | Jeune — évaluer la stabilité avant tout projet client |
| Briques | Blocks (plugins, inline, fields) | Composants SDC + code components |
| UX | Drag & drop par section | Édition visuelle complète in-place |
| Theming | Twig classique | SDC + design tokens |

**Décision projet** : nouveau site vitrine longue durée → évaluer XB (vérifier la version stable du
moment et la compatibilité contrib) ; projet client à livrer maintenant ou refonte existante →
Layout Builder reste le choix sûr. Dans tous les cas, **construire les composants en SDC dès
aujourd'hui** : ils sont réutilisables tels quels dans XB demain (voir `drupal-sdc`).
