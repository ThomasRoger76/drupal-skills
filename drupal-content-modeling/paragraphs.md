---
name: drupal-content-modeling — paragraphs
description: Guide complet du module Paragraphs pour Drupal 8-11+. Configuration des types de paragraphes, Paragraphs Behaviors, paragraphes imbriqués, performance N+1, theming Twig, accès programmatique aux paragraphes.
---

# Paragraphs — Guide Complet

## Architecture de Base

```
Node (article)
  └── field_contenu (champ Entity Reference Revisions → Paragraph)
       ├── Paragraph type: hero_image
       │     ├── field_titre (string)
       │     └── field_image (media reference)
       ├── Paragraph type: texte_libre
       │     └── field_body (text_with_summary)
       └── Paragraph type: section_trois_colonnes
             ├── field_colonne_1 (entity_reference_revisions → sous-paragraphs)
             ├── field_colonne_2 (entity_reference_revisions → sous-paragraphs)
             └── field_colonne_3 (entity_reference_revisions → sous-paragraphs)
```

---

## Configuration d'un Champ Paragraphs

```yaml
# Ajouter un champ Paragraphs à un Content Type depuis le code
# config/install/field.field.node.article.field_contenu.yml

langcode: fr
status: true
id: node.article.field_contenu
field_name: field_contenu
entity_type: node
bundle: article
label: 'Contenu de la page'
description: ''
required: false
translatable: true
field_type: entity_reference_revisions
settings:
  handler: 'default:paragraph'
  handler_settings:
    target_bundles:
      hero_image: hero_image
      texte_libre: texte_libre
      section_trois_colonnes: section_trois_colonnes
    target_bundles_drag_drop:
      hero_image:
        enabled: true
        weight: 0
      texte_libre:
        enabled: true
        weight: 1
```

---

## Paragraphs Behaviors — Comportements par Type

Les Behaviors ajoutent des options de configuration (layout, couleur de fond, taille) directement dans le formulaire d'édition, sans champs supplémentaires.

```php
<?php
// src/Plugin/paragraphs/Behavior/SectionBehavior.php
namespace Drupal\mon_module\Plugin\paragraphs\Behavior;

use Drupal\Core\Form\FormStateInterface;
use Drupal\paragraphs\Annotation\ParagraphsBehavior;
use Drupal\paragraphs\Entity\Paragraph;
use Drupal\paragraphs\Entity\ParagraphsType;
use Drupal\paragraphs\ParagraphsBehaviorBase;

/**
 * Configure la mise en page d'une section (pleine largeur, boxed, coloré).
 *
 * @ParagraphsBehavior(
 *   id = "mon_module_section_behavior",
 *   label = @Translation("Mise en page de section"),
 *   description = @Translation("Options de layout pour les sections."),
 *   weight = 0,
 * )
 */
// D11 : utiliser #[ParagraphsBehavior(...)] attribute PHP

class SectionBehavior extends ParagraphsBehaviorBase {

  /**
   * Formulaire des options de behavior (affiché dans le panneau summary).
   */
  public function buildBehaviorForm(
    Paragraph $paragraph,
    array &$form,
    FormStateInterface $form_state
  ): array {
    $form = parent::buildBehaviorForm($paragraph, $form, $form_state);

    $form['layout'] = [
      '#type' => 'select',
      '#title' => $this->t('Mise en page'),
      '#options' => [
        'full' => $this->t('Pleine largeur'),
        'boxed' => $this->t('Contenu centré (max-width)'),
        'narrow' => $this->t('Étroit (article)'),
      ],
      '#default_value' => $paragraph->getBehaviorSetting($this->getPluginId(), 'layout', 'boxed'),
    ];

    $form['couleur_fond'] = [
      '#type' => 'select',
      '#title' => $this->t('Couleur de fond'),
      '#options' => [
        'blanc' => $this->t('Blanc'),
        'gris-clair' => $this->t('Gris clair'),
        'primaire' => $this->t('Couleur primaire'),
        'sombre' => $this->t('Sombre'),
      ],
      '#default_value' => $paragraph->getBehaviorSetting($this->getPluginId(), 'couleur_fond', 'blanc'),
    ];

    return $form;
  }

  /**
   * Valider le formulaire du behavior.
   */
  public function validateBehaviorForm(
    Paragraph $paragraph,
    array &$form,
    FormStateInterface $form_state
  ): void {
    // Validation custom si nécessaire
  }

  /**
   * Ce behavior s'applique-t-il à ce type de paragraphe ?
   */
  public static function isApplicable(ParagraphsType $paragraphs_type): bool {
    // Uniquement pour le type 'section'
    return in_array($paragraphs_type->id(), ['section', 'hero', 'section_trois_colonnes']);
  }

  /**
   * Résumé affiché dans le formulaire d'édition (accordion header).
   */
  public function settingsSummary(Paragraph $paragraph): array {
    $summary = [];
    $layout = $paragraph->getBehaviorSetting($this->getPluginId(), 'layout', 'boxed');
    $couleur = $paragraph->getBehaviorSetting($this->getPluginId(), 'couleur_fond', 'blanc');
    $summary[] = $this->t('Layout: @layout | Fond: @couleur', [
      '@layout' => $layout,
      '@couleur' => $couleur,
    ]);
    return $summary;
  }

  /**
   * Prétraitement de la view — injecter les variables behavior dans Twig.
   */
  public function preprocess(array &$variables, $hook, array $info): void {
    /** @var \Drupal\paragraphs\Entity\Paragraph $paragraph */
    $paragraph = $variables['paragraph'];
    $plugin_id = $this->getPluginId();

    $variables['behavior_settings'] = [
      'layout' => $paragraph->getBehaviorSetting($plugin_id, 'layout', 'boxed'),
      'couleur_fond' => $paragraph->getBehaviorSetting($plugin_id, 'couleur_fond', 'blanc'),
    ];
  }
}
```

---

## Accès Programmatique aux Paragraphes

```php
// Charger les paragraphes d'un nœud
$node = Node::load($nid);

// Accès au champ paragraphs
$paragraphs_field = $node->get('field_contenu');

// Itérer sur les paragraphes
foreach ($paragraphs_field as $item) {
  /** @var \Drupal\paragraphs\Entity\Paragraph $paragraph */
  $paragraph = $item->entity;

  if (!$paragraph) {
    continue;  // Paragraphe supprimé
  }

  $type = $paragraph->getType();  // ex: 'hero_image', 'texte_libre'

  // Accéder aux champs selon le type
  if ($type === 'texte_libre') {
    $body = $paragraph->get('field_body')->value;
    $format = $paragraph->get('field_body')->format;
  }

  // Lire un behavior setting
  $layout = $paragraph->getBehaviorSetting('mon_module_section_behavior', 'layout', 'boxed');
}

// ⚠️ PROBLÈME N+1 — charger les paragraphes UN PAR UN
// Chaque $item->entity déclenche une requête DB séparée
// Pour 10 paragraphes = 10 + 1 requêtes

// ✅ SOLUTION — charger tous les paragraphes en une fois
$paragraph_ids = array_column(
  $node->get('field_contenu')->getValue(),
  'target_id'
);

if ($paragraph_ids) {
  // Batch loading — 1 seule requête pour tous les paragraphes
  $paragraphs = \Drupal::entityTypeManager()
    ->getStorage('paragraph')
    ->loadMultiple($paragraph_ids);
}
```

---

## Paragraphes Imbriqués — Précautions

```php
// ⚠️ ATTENTION aux paragraphes imbriqués profonds
// Niveau 1 : node → paragraphs (1 requête batch)
// Niveau 2 : paragraph → sous-paragraphs (N requêtes si pas batché)
// Niveau 3 : sous-paragraph → sous-sous-paragraphs (N² requêtes)

// ✅ Pattern correct pour les paragraphes imbriqués
function charger_paragraphes_imbriques(array $paragraph_ids): array {
  $paragraphs = \Drupal::entityTypeManager()
    ->getStorage('paragraph')
    ->loadMultiple($paragraph_ids);

  // Collecter les IDs des sous-paragraphes
  $sous_ids = [];
  foreach ($paragraphs as $paragraph) {
    if ($paragraph->hasField('field_colonnes')) {
      foreach ($paragraph->get('field_colonnes') as $item) {
        if ($item->target_id) {
          $sous_ids[] = $item->target_id;
        }
      }
    }
  }

  // Charger tous les sous-paragraphes en batch
  if ($sous_ids) {
    $sous_paragraphs = \Drupal::entityTypeManager()
      ->getStorage('paragraph')
      ->loadMultiple($sous_ids);
  }

  return $paragraphs;
}
```

---

## Templates Twig pour Paragraphs

```
# Hiérarchie des suggestions de templates Paragraphs
paragraph.html.twig                                    (le plus général)
paragraph--TYPE.html.twig                              (par type)
paragraph--TYPE--view-mode.html.twig                   (par type + view mode)
```

```twig
{# templates/paragraph/paragraph--section.html.twig #}

{%
  set classes = [
    'paragraph',
    'paragraph--type--' ~ paragraph.bundle|clean_class,
    paragraph.isPublished() ? 'paragraph--view-mode--' ~ view_mode|clean_class,
    'layout--' ~ behavior_settings.layout|default('boxed')|clean_class,
    'couleur-fond--' ~ behavior_settings.couleur_fond|default('blanc')|clean_class,
  ]
%}

<section{{ attributes.addClass(classes) }}>
  <div class="section__inner">
    {% if content.field_titre is not empty %}
      <h2 class="section__titre">{{ content.field_titre }}</h2>
    {% endif %}

    {% for item in content.field_colonnes %}
      {# Accéder à chaque sous-paragraphe #}
      <div class="section__colonne">
        {{ item }}
      </div>
    {% endfor %}
  </div>
</section>

{# Variables disponibles :
   paragraph         : entité Paragraph
   content           : tableau de champs rendus
   attributes        : objet Attribute HTML
   view_mode         : mode de rendu ('default', 'preview', etc.)
   behavior_settings : variables injectées par ParagraphsBehavior::preprocess()
   elements          : tableau brut des éléments rendus
#}
```

---

## Créer un Paragraphe Programmatiquement

```php
use Drupal\paragraphs\Entity\Paragraph;
use Drupal\node\Entity\Node;

// 1. Créer le paragraphe
$paragraph = Paragraph::create([
  'type' => 'texte_libre',
  'field_body' => [
    'value' => '<p>Contenu du paragraphe</p>',
    'format' => 'basic_html',
  ],
]);
$paragraph->save();

// 2. L'attacher au nœud
$node = Node::load($nid);
$node->get('field_contenu')->appendItem([
  'target_id' => $paragraph->id(),
  'target_revision_id' => $paragraph->getRevisionId(),
]);
$node->save();

// Ajouter un behavior setting avant save
$paragraph->setBehaviorSettings('mon_module_section_behavior', [
  'layout' => 'full',
  'couleur_fond' => 'primaire',
]);
$paragraph->save();
```

---

## Performance — Varnish & Cache avec Paragraphs

```php
// Dans un preprocess_node avec Paragraphs
function mon_theme_preprocess_node(&$variables): void {
  $node = $variables['node'];

  // Collecter les cache tags de tous les paragraphes
  $tags = $node->getCacheTags();

  foreach ($node->get('field_contenu') as $item) {
    $paragraph = $item->entity;
    if ($paragraph) {
      $tags = \Drupal\Core\Cache\Cache::mergeTags($tags, $paragraph->getCacheTags());
    }
  }

  // Ajouter les tags au render array
  $variables['#cache']['tags'] = $tags;
}
```
