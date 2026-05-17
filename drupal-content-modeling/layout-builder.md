---
name: drupal-content-modeling — layout builder
description: Drupal Layout Builder - activation, configuration programmatique, Custom Layout Sections, Layout Builder vs Paragraphs décision, et accès à la config de layout depuis PHP.
---

# Layout Builder — Référence Complète

## Activation sur un Content Type

```php
// Activer Layout Builder programmatiquement sur le Content Type 'article'
use Drupal\Core\Entity\Display\EntityViewDisplayInterface;

$display = \Drupal::entityTypeManager()
  ->getStorage('entity_view_display')
  ->load('node.article.full');

if ($display instanceof EntityViewDisplayInterface) {
  $display->enableLayoutBuilder()
    ->setOverridable(TRUE)   // TRUE = chaque nœud peut avoir son propre layout
    ->save();                // FALSE = layout unique pour tout le type
}

// Vérifier si Layout Builder est actif
$is_enabled = $display->isLayoutBuilderEnabled();
$is_overridable = $display->isOverridable();
```

```bash
# Via drush
drush php:eval "
\$display = \Drupal::entityTypeManager()
  ->getStorage('entity_view_display')
  ->load('node.article.full');
\$display->enableLayoutBuilder()->setOverridable(TRUE)->save();
echo 'Layout Builder activé.';
"
```

---

## Accéder au Layout d'un Nœud

```php
use Drupal\layout_builder\LayoutEntityHelperTrait;

class MonService {
  use LayoutEntityHelperTrait;

  public function getNodeLayout(\Drupal\node\NodeInterface $node): array {
    // Obtenir le section storage pour ce nœud
    $section_storage = $this->getSectionStorageForEntity($node);

    if (!$section_storage) {
      // Pas de layout personnalisé → utilise le layout du display par défaut
      return [];
    }

    $layout_info = [];
    foreach ($section_storage->getSections() as $delta => $section) {
      $layout_info[$delta] = [
        'layout_id' => $section->getLayoutId(),
        'config' => $section->getLayoutSettings(),
        'components' => [],
      ];

      foreach ($section->getComponents() as $uuid => $component) {
        $layout_info[$delta]['components'][$uuid] = [
          'plugin_id' => $component->getPluginId(),
          'region' => $component->getRegion(),
          'weight' => $component->getWeight(),
          'config' => $component->get('configuration'),
        ];
      }
    }

    return $layout_info;
  }

  public function nodeHasCustomLayout(\Drupal\node\NodeInterface $node): bool {
    return $node->hasField('layout_builder__layout')
      && !$node->get('layout_builder__layout')->isEmpty();
  }
}
```

---

## Ajouter un Bloc à un Layout Programmatiquement

```php
use Drupal\layout_builder\Section;
use Drupal\layout_builder\SectionComponent;

// Charger le nœud et son section storage
$node = \Drupal\node\Entity\Node::load(42);

$section_storage = \Drupal::service('plugin.manager.layout_builder.section_storage')
  ->load('overrides', ['entity' => $node->getTypedData()]);

if (!$section_storage) {
  // Pas encore de layout custom — initialiser depuis le display par défaut
  $display = \Drupal::entityTypeManager()
    ->getStorage('entity_view_display')
    ->load('node.' . $node->bundle() . '.full');

  // Copier les sections par défaut
  $sections = $display->getSections();
  foreach ($sections as $section) {
    $section_storage->appendSection($section);
  }
}

// Ajouter un bloc à la première section, région 'content'
$first_section = $section_storage->getSection(0);

$component = SectionComponent::fromArray([
  'id' => \Drupal\Component\Utility\NestedArray::getValue([], []),
  'uuid' => \Drupal::service('uuid')->generate(),
  'region' => 'content',
  'weight' => 0,
  'configuration' => [
    'id' => 'basic_block:uuid-du-custom-block',  // ou 'field_block:node:article:title'
    'label' => 'Mon bloc',
    'label_display' => FALSE,
    'view_mode' => 'full',
    'context_mapping' => [],
  ],
  'additional' => [],
]);

$first_section->appendComponent($component);
$section_storage->save();
```

---

## Layout Builder vs Paragraphs — Checklist Finale

```
Utiliser Layout Builder quand :
  ✅ L'éditeur doit VOIR visuellement où il place les composants
  ✅ Chaque nœud peut avoir une mise en page DIFFÉRENTE
  ✅ Les blocs sont réutilisables entre plusieurs pages
  ✅ Les éditeurs sont expérimentés (interface plus complexe)
  ✅ Site de contenu marketing (landing pages, pages de fonctionnalités)

Utiliser Paragraphs quand :
  ✅ La mise en page est définie par les développeurs (types fixes)
  ✅ Les éditeurs remplissent des "composants" prédéfinis
  ✅ Le contenu est structuré (articles avec sections normalisées)
  ✅ Besoin de migration Migrate API (meilleur support)
  ✅ Site multilingue (meilleur support translations)
  ✅ API-first / headless (normalisation JSON:API native)

⚠️ Éviter :
  ❌ Utiliser les deux ensemble (complexité excessive)
  ❌ Layout Builder sur du contenu haute fréquence (stockage par nœud)
  ❌ Paragraphs imbriqués > 3 niveaux
  ❌ Changer d'approche après déploiement (migration complexe)
```

---

## Commandes Layout Builder

```bash
# Vérifier si Layout Builder est actif sur un display
drush php:eval "
\$display = \Drupal::entityTypeManager()
  ->getStorage('entity_view_display')
  ->load('node.article.full');
echo 'Layout Builder: ' . (\$display->isLayoutBuilderEnabled() ? 'OUI' : 'NON') . PHP_EOL;
echo 'Overridable: ' . (\$display->isOverridable() ? 'OUI' : 'NON') . PHP_EOL;
"

# Désactiver Layout Builder
drush php:eval "
\$display = \Drupal::entityTypeManager()
  ->getStorage('entity_view_display')
  ->load('node.article.full');
\$display->disableLayoutBuilder()->save();
echo 'Désactivé.';
"

# Voir les sections d'un nœud spécifique
drush php:eval "
\$node = \Drupal::entityTypeManager()->getStorage('node')->load(1);
if (\$node->hasField('layout_builder__layout')) {
  echo count(\$node->get('layout_builder__layout')->getSections()) . ' section(s)' . PHP_EOL;
}
"
```
