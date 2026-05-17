---
name: drupal-content-modeling — block content types
description: Créer des Custom Block Types (bundles de blocs avec champs) dans Drupal - configuration, placement, accès programmatique, et patterns avec Layout Builder.
---

# Custom Block Types — Référence Complète

## Qu'est-ce qu'un Custom Block Type

```
Block Type (bundle de bloc) ≠ Block Plugin (PHP)
  
Block Type (block_content) :
  → Entité content Drupal avec champs configurables
  → Créé dans l'UI : /admin/structure/block/block-content/types
  → Placé dans les régions via /admin/structure/block
  → Réutilisable (1 bloc dans plusieurs régions/pages)
  → Translatable
  → Révisions supportées

Block Plugin (PHP @Block / #[Block]) :
  → Plugin PHP avec logique custom
  → Non stocké en DB (pas de révisions, pas d'éditeur UI)
  → Pour les blocs dynamiques (contenu calculé)
```

---

## Créer un Block Type via YAML

```yaml
# config/install/block_content.type.appel_a_action.yml
langcode: fr
status: true
id: appel_a_action
label: 'Appel à action (CTA)'
description: 'Bloc avec titre, texte, et bouton d''action.'
revision: true     # Activer les révisions
```

```yaml
# Champ Titre custom (en plus du label natif)
# config/install/field.storage.block_content.field_cta_titre.yml
langcode: fr
status: true
id: block_content.field_cta_titre
field_name: field_cta_titre
entity_type: block_content
type: string
cardinality: 1

# config/install/field.field.block_content.appel_a_action.field_cta_titre.yml
langcode: fr
status: true
id: block_content.appel_a_action.field_cta_titre
field_name: field_cta_titre
entity_type: block_content
bundle: appel_a_action
label: 'Titre'
required: true
translatable: true
```

---

## Créer et Placer un Bloc Custom Programmatiquement

```php
use Drupal\block_content\Entity\BlockContent;
use Drupal\block\Entity\Block;

// 1. Créer l'entité de contenu du bloc
$block_content = BlockContent::create([
  'type' => 'appel_a_action',
  'info' => 'CTA — Page d\'accueil',  // Label admin
  'langcode' => 'fr',
  'field_cta_titre' => 'Découvrez notre offre',
  'field_cta_texte' => [
    'value' => '<p>Profitez de notre offre exclusive.</p>',
    'format' => 'basic_html',
  ],
  'field_cta_url' => ['uri' => 'internal:/offres', 'title' => 'Voir les offres'],
]);
$block_content->save();

// 2. Placer le bloc dans une région (via entité Block)
$block = Block::create([
  'id' => 'appel_a_action_accueil',
  'plugin' => 'block_content:' . $block_content->uuid(),
  'region' => 'content',
  'theme' => 'mon_theme',
  'settings' => [
    'label' => 'Appel à action',
    'label_display' => FALSE,
  ],
  'visibility' => [
    'request_path' => [
      'id' => 'request_path',
      'negate' => FALSE,
      'pages' => '<front>',
    ],
  ],
]);
$block->save();
```

---

## Accéder à un Block Content depuis PHP

```php
use Drupal\block_content\Entity\BlockContent;

// Charger par UUID (stable entre environnements)
$uuid = 'uuid-du-bloc';
$blocks = \Drupal::entityTypeManager()
  ->getStorage('block_content')
  ->loadByProperties(['uuid' => $uuid]);
$block = reset($blocks);

// Charger par label (info)
$blocks = \Drupal::entityTypeManager()
  ->getStorage('block_content')
  ->loadByProperties(['info' => 'Mon Bloc CTA', 'type' => 'appel_a_action']);

// Requête plus complexe
$bids = \Drupal::entityQuery('block_content')
  ->condition('type', 'appel_a_action')
  ->condition('status', 1)
  ->sort('changed', 'DESC')
  ->accessCheck(TRUE)
  ->execute();
$blocks = BlockContent::loadMultiple($bids);

// Accéder aux champs
$block = BlockContent::load($bid);
$titre = $block->get('field_cta_titre')->value;
$texte = $block->get('field_cta_texte')->value;
$url = $block->get('field_cta_url')->uri;
```

---

## Template Twig pour Block Content

```
# Suggestions de templates :
block.html.twig                               (tous les blocs)
block--block-content.html.twig               (tous les block_content)
block--block-content--BUNDLE.html.twig       (par type de bundle)
block--PLUGIN-ID.html.twig                   (ID du plugin bloc)

# Exemple pour type 'appel_a_action' :
block--block-content--appel-a-action.html.twig
```

```twig
{# block--block-content--appel-a-action.html.twig #}
<div{{ attributes.addClass('cta-block') }}>
  {% block content %}
    <h2 class="cta-block__titre">{{ content.field_cta_titre }}</h2>
    <div class="cta-block__texte">{{ content.field_cta_texte }}</div>
    {% if content.field_cta_url %}
      <div class="cta-block__action">{{ content.field_cta_url }}</div>
    {% endif %}
  {% endblock %}
</div>
```

---

## Block Content dans Layout Builder

Layout Builder peut utiliser des Block Content comme blocs réutilisables :

```
Layout Builder → Add block → Content block → Choisir le type → Créer ou utiliser existant

Configuration :
  ├── "Create new block" → crée un BlockContent entity + l'insère
  ├── "Reuse existing block" → utilise un BlockContent existant (réutilisable)
  └── "Create per-revision copy" → copie locale non réutilisable
```

---

## Traduction d'un Block Content

```php
// Traduire un bloc existant
$block = BlockContent::load($bid);
if (!$block->hasTranslation('en')) {
  $translated = $block->addTranslation('en', [
    'info' => 'CTA — Homepage',
    'field_cta_titre' => 'Discover our offer',
  ]);
  $block->save();
}

// Accéder à la traduction courante
$block = \Drupal::service('entity.repository')
  ->getTranslationFromContext($block);
```
