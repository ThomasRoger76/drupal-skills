---
name: drupal-sdc — SDC dans Layout Builder
description: Utiliser les Single Directory Components comme blocs réutilisables dans Layout Builder - création d'un Block plugin basé sur SDC, props depuis la configuration du bloc.
---

# SDC + Layout Builder — Référence Complète

## Architecture de l'Intégration

```
SDC Component (card.twig + card.component.yml)
       ↓
Block Plugin PHP (#[Block]) qui rend le composant SDC
       ↓
Layout Builder → Add block → Choisir le Block Plugin → Configurer les props
       ↓
Rendu final : layout_builder appelle le Block, qui rend le SDC
```

---

## Créer un Block Plugin basé sur un SDC

```php
<?php
// src/Plugin/Block/CardBlock.php
namespace Drupal\mon_module\Plugin\Block;

use Drupal\Core\Block\BlockBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Block basé sur le composant SDC 'card'.
 *
 * @Block(
 *   id = "mon_module_card",
 *   admin_label = @Translation("Carte (SDC)"),
 *   category = @Translation("Composants"),
 * )
 */
// D11 : #[Block(id: "mon_module_card", admin_label: new TranslatableMarkup("Carte"), ...)]
class CardBlock extends BlockBase {

  /**
   * Formulaire de configuration — permet à l'éditeur de remplir les props SDC.
   */
  public function blockForm(array $form, FormStateInterface $form_state): array {
    $form = parent::blockForm($form, $form_state);
    $config = $this->getConfiguration();

    $form['title'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Titre'),
      '#default_value' => $config['title'] ?? '',
      '#required' => TRUE,
    ];

    $form['url'] = [
      '#type' => 'url',
      '#title' => $this->t('URL de destination'),
      '#default_value' => $config['url'] ?? '',
    ];

    $form['summary'] = [
      '#type' => 'textarea',
      '#title' => $this->t('Résumé'),
      '#default_value' => $config['summary'] ?? '',
      '#rows' => 3,
    ];

    $form['variant'] = [
      '#type' => 'select',
      '#title' => $this->t('Variante'),
      '#options' => [
        'default'  => $this->t('Standard'),
        'featured' => $this->t('Mis en avant'),
        'compact'  => $this->t('Compact'),
      ],
      '#default_value' => $config['variant'] ?? 'default',
    ];

    return $form;
  }

  public function blockSubmit(array $form, FormStateInterface $form_state): void {
    $this->setConfigurationValue('title', $form_state->getValue('title'));
    $this->setConfigurationValue('url', $form_state->getValue('url'));
    $this->setConfigurationValue('summary', $form_state->getValue('summary'));
    $this->setConfigurationValue('variant', $form_state->getValue('variant'));
  }

  /**
   * Rendu — utiliser le composant SDC directement.
   */
  public function build(): array {
    $config = $this->getConfiguration();

    // Utiliser le render array de type 'component' pour SDC
    return [
      '#type' => 'component',
      '#component' => 'mon_theme:card',   // {theme}:{component-name}
      '#props' => [
        'title'   => $config['title'] ?? '',
        'url'     => $config['url'] ?? '#',
        'summary' => $config['summary'] ?? '',
        'variant' => $config['variant'] ?? 'default',
      ],
      // Cache — invalider quand le bloc change
      '#cache' => [
        'tags' => $this->getCacheTags(),
        'contexts' => $this->getCacheContexts(),
        'max-age' => $this->getCacheMaxAge(),
      ],
    ];
  }

  public function defaultConfiguration(): array {
    return [
      'title' => '',
      'url' => '',
      'summary' => '',
      'variant' => 'default',
    ];
  }
}
```

---

## Utilisation dans Layout Builder

```
/admin/structure/types/manage/page/display → Layout Builder activé

Page → Onglet Layout → Add block
  → Rechercher "Carte (SDC)"
  → Remplir les props : Titre, URL, Résumé, Variante
  → Sauvegarder → Le composant SDC est rendu

Avantages vs Block Content type :
  ✅ Props typées et validées par SDC
  ✅ CSS/JS automatiquement attachés par SDC
  ✅ Storybook disponible pour preview
  ✅ Pas de stockage DB (plus léger qu'un block_content)
```

---

## SDC avec Entity-Based Props (entité Drupal)

```php
// Pour passer une entité Drupal complète comme prop SDC
public function build(): array {
  $config = $this->getConfiguration();
  $nid = $config['node_reference'] ?? NULL;

  if (!$nid) return [];

  $node = \Drupal::entityTypeManager()->getStorage('node')->load($nid);
  if (!$node || !$node->access('view')) return [];

  return [
    '#type' => 'component',
    '#component' => 'mon_theme:article-card',
    '#props' => [
      'title'     => $node->getTitle(),
      'url'       => $node->toUrl()->toString(),
      'summary'   => $node->get('body')->summary ?: '',
      'image_url' => $this->getNodeImageUrl($node),
    ],
    '#cache' => [
      'tags' => $node->getCacheTags(),
      'contexts' => ['user.permissions'],
    ],
  ];
}
```

---

## SDC Section Plugin pour Layout Builder

Pour créer des layouts SDC-based :

```php
<?php
// src/Plugin/Layout/CardGridLayout.php
use Drupal\Core\Layout\LayoutBase;

/**
 * @Layout(
 *   id = "mon_module_card_grid",
 *   label = @Translation("Grille de cartes SDC"),
 *   category = @Translation("Mon Module"),
 *   regions = {
 *     "carte_1" = { "label" = @Translation("Carte 1") },
 *     "carte_2" = { "label" = @Translation("Carte 2") },
 *     "carte_3" = { "label" = @Translation("Carte 3") },
 *   }
 * )
 */
class CardGridLayout extends LayoutBase {

  public function build(array $regions): array {
    return [
      '#type' => 'component',
      '#component' => 'mon_theme:card-grid',
      '#props' => ['colonnes' => 3],
      '#slots' => [
        'carte_1' => $regions['carte_1'] ?? [],
        'carte_2' => $regions['carte_2'] ?? [],
        'carte_3' => $regions['carte_3'] ?? [],
      ],
    ];
  }
}
```

---

## Debug SDC dans Layout Builder

```bash
# Lister les composants SDC disponibles
drush php:eval "
foreach (\Drupal::service('sdc.component_registry')->getAllComponents() as \$id => \$c) {
  echo \$id . PHP_EOL;
}
"

# Vérifier qu'un block plugin est découvert
drush php:eval "
\$manager = \Drupal::service('plugin.manager.block');
\$defs = \$manager->getDefinitions();
echo isset(\$defs['mon_module_card']) ? 'Block plugin trouvé' : 'Block plugin ABSENT';
"

# Vider le cache après modification d'un block plugin
drush cr
```
