---
name: drupal-content-modeling — nodes et content types
description: Configurer les Content Types Drupal (nodes) - champs, display modes, workflow, path aliases, révisions, et choix entre node vs custom entity.
---

# Nodes & Content Types — Référence Complète

## Quand Utiliser un Node

```
Node = entité Drupal avec :
  ✅ URL propre (/mon-article) + alias Pathauto
  ✅ Révisions (historique des versions)
  ✅ Workflow éditorial (draft → published → archived)
  ✅ Traduction multilingue native
  ✅ Menu links automatiques
  ✅ Breadcrumbs natifs
  ✅ Promoted to front page
  ✅ Access control (published/unpublished)
  ✅ Comments (si module comments actif)
  ✅ Fichiers de template Twig dédiés (node--TYPE.html.twig)
```

---

## Créer un Content Type en Code

```yaml
# config/install/node.type.evenement.yml
langcode: fr
status: true
dependencies:
  module:
    - node
id: evenement
label: Événement
description: 'Événement du site (conférence, atelier, webinaire)'
help: ''
new_revision: true              # Créer une révision à chaque sauvegarde
preview_mode: 1                 # 1 = preview optionnel, 0 = désactivé
display_submitted: false        # Masquer "Soumis par X le Y"
```

```yaml
# config/install/core.base_field_override.node.evenement.status.yml
# Publié par défaut (ou non)
langcode: fr
status: true
id: node.evenement.status
field_name: status
entity_type: node
bundle: evenement
default_value:
  - value: 0     # 0 = brouillon par défaut, 1 = publié par défaut
```

---

## Ajouter des Champs via Drush

```bash
# Créer un champ texte court sur le content type 'evenement'
# (Via l'UI : /admin/structure/types/manage/evenement/fields/add-field)

# Via config YAML — field storage
# config/install/field.storage.node.field_lieu.yml
field_name: field_lieu
entity_type: node
type: string
cardinality: 1

# config/install/field.field.node.evenement.field_lieu.yml
field_name: field_lieu
entity_type: node
bundle: evenement
label: 'Lieu de l''événement'
required: false
```

---

## Display Modes — View Modes

```bash
# Lister les view modes disponibles
drush php:eval "
\$modes = \Drupal::entityTypeManager()->getStorage('entity_view_mode')->loadMultiple();
foreach (\$modes as \$id => \$mode) {
  if (str_starts_with(\$id, 'node.')) {
    echo \$id . ': ' . \$mode->label() . PHP_EOL;
  }
}
"

# View modes standards :
# node.full      → page complète (canonical)
# node.teaser    → résumé (listes, Views)
# node.rss       → flux RSS
# node.search_index → pour l'indexation Search API
# node.search_result → résultats de recherche
```

```php
// Rendre un nœud dans un view mode spécifique
$node = \Drupal\node\Entity\Node::load(42);
$view = \Drupal::entityTypeManager()->getViewBuilder('node')->view($node, 'teaser');
$rendered = \Drupal::service('renderer')->renderRoot($view);
```

---

## Accès Programmatique aux Nodes

```php
use Drupal\node\Entity\Node;

// Créer un nœud
$node = Node::create([
  'type' => 'evenement',
  'title' => 'Drupal Camp France 2026',
  'status' => 1,   // 1 = publié, 0 = brouillon
  'langcode' => 'fr',
  'uid' => \Drupal::currentUser()->id(),
  'field_lieu' => 'Paris, France',
  'field_date_evenement' => [
    'value' => '2026-10-15T09:00:00',
    'end_value' => '2026-10-16T18:00:00',
  ],
  'field_tags' => [
    ['target_id' => 42],
    ['target_id' => 43],
  ],
]);
$node->save();

// Charger et modifier
$node = Node::load(42);
$node->setTitle('Nouveau titre');
$node->setPublished();   // ou $node->setUnpublished()
$node->save();

// Requête avec conditions
$nids = \Drupal::entityQuery('node')
  ->condition('type', 'evenement')
  ->condition('status', 1)
  ->condition('field_date_evenement', date('Y-m-d\TH:i:s'), '>=')
  ->sort('field_date_evenement', 'ASC')
  ->range(0, 10)
  ->accessCheck(TRUE)
  ->execute();
$events = Node::loadMultiple($nids);

// Supprimer
$node->delete();
\Drupal::entityTypeManager()->getStorage('node')->delete($nodes);
```

---

## Révisions Drupal

```php
// Activer les révisions sur un nœud lors de la sauvegarde
$node->setNewRevision(TRUE);
$node->setRevisionLogMessage('Correction typographique');
$node->setRevisionCreationTime(\Drupal::time()->getRequestTime());
$node->setRevisionUserId(\Drupal::currentUser()->id());
$node->save();

// Charger une révision spécifique
$revision_id = 15;
$node_revision = \Drupal::entityTypeManager()
  ->getStorage('node')
  ->loadRevision($revision_id);

// Lister les révisions d'un nœud
$revision_ids = \Drupal::entityTypeManager()
  ->getStorage('node')
  ->revisionIds($node);
```

---

## Workflow Editorial (Content Moderation)

```bash
# Activer Content Moderation
drush en content_moderation workflows -y

# Créer un workflow (via UI ou config YAML)
# /admin/config/workflow/workflows/add
```

```yaml
# config/install/workflows.workflow.editorial.yml
langcode: fr
status: true
id: editorial
label: 'Workflow Editorial'
type: content_moderation
type_settings:
  states:
    draft:
      label: Brouillon
      published: false
      default_revision: false
      weight: 0
    published:
      label: Publié
      published: true
      default_revision: true
      weight: 1
    archived:
      label: Archivé
      published: false
      default_revision: false
      weight: 2
  transitions:
    create_new_draft:
      label: 'Créer un nouveau brouillon'
      from: [draft, published]
      to: draft
      weight: 0
    publish:
      label: Publier
      from: [draft]
      to: published
      weight: 1
    archive:
      label: Archiver
      from: [published]
      to: archived
      weight: 2
  entity_types:
    node:
      - evenement
      - article
```

```php
// Changer l'état de modération programmatiquement
$node = Node::load(42);
$node->set('moderation_state', 'published');
$node->save();

// Obtenir l'état actuel
$state = $node->get('moderation_state')->value;
// → 'draft', 'published', 'archived'
```

---

## Path Aliases et Pathauto

```bash
composer require drupal/pathauto
drush en pathauto -y
```

```yaml
# Pattern d'alias pour les événements
# config/install/pathauto.pattern.evenements.yml
langcode: fr
status: true
id: evenements
label: 'Événements'
type: 'canonical_entities:node'
pattern: 'evenements/[node:title]'
selection_criteria:
  bundle:
    id: condition.bundle
    bundles:
      evenement: evenement
```

```php
// Obtenir l'alias d'un nœud
$path_alias_manager = \Drupal::service('path_alias.manager');
$alias = $path_alias_manager->getAliasByPath('/node/' . $node->id());
// → '/evenements/drupal-camp-france-2026'

// Générer ou regénérer l'alias d'un nœud
\Drupal::service('pathauto.generator')->createEntityAlias($node, 'insert');
```
