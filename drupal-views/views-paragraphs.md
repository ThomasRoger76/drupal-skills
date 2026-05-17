---
name: drupal-views — paragraphs dans Views
description: Afficher et filtrer des Paragraphs dans les Views Drupal - relations, field handlers, templates, et patterns de performance.
---

# Views + Paragraphs — Référence Complète

## Le Problème Fondamental

Les Paragraphs ne sont **pas** directement listables dans une View de type "Content" — ils sont attachés à des nœuds via `entity_reference_revisions`. Il faut choisir une stratégie selon le besoin.

---

## Stratégie 1 — Afficher les Paragraphs d'un Nœud via Relationship

La plus courante : créer une View de **nœuds** et accéder aux paragraphs via une relation.

```
View → Source : Content (node)
     → Relationship : ajouter "field_contenu" (le champ paragraphs)
                     → "Require this relationship" : OFF (LEFT JOIN)
     → Fields : ajouter les champs du paragraphs via la relation
     → Filter : type de paragraph (si besoin)
```

**Configuration YAML :**
```yaml
# Dans views.view.articles_avec_paragraphs.yml
relationships:
  field_contenu_target_revision_id:
    id: field_contenu_target_revision_id
    table: node__field_contenu
    field: field_contenu_target_revision_id
    relationship: none
    label: 'Paragraphes du contenu'
    required: false    # LEFT JOIN — montre les nœuds même sans paragraphes
    plugin_id: entity_reference_revisions
```

---

## Stratégie 2 — View basée sur les Paragraphs directement

Pour lister des paragraphs d'un type spécifique à travers tous les nœuds :

```
Views → Add new view → Show : Paragraphs
→ Filter : Type de paragraphe = "hero_image"
→ Relationship : "Node from Paragraphs" (pour accéder aux champs du nœud parent)
```

**Pré-requis :** le module `paragraphs` expose les paragraphs à Views via `hook_views_data()`.

---

## Stratégie 3 — hook_views_data() pour Table Paragraphs Custom

Pour accéder directement à la table `paragraphs_item_field_data` :

```php
function mon_module_views_data_alter(array &$data): void {
  // Relation inverse : depuis un nœud, accéder à ses paragraphs
  if (isset($data['node_field_data'])) {
    $data['node_field_data']['paragraphs_du_noeud'] = [
      'title' => t('Paragraphes du nœud'),
      'help' => t('Accède aux paragraphes attachés à ce nœud.'),
      'relationship' => [
        'id' => 'standard',
        'base' => 'paragraphs_item_field_data',
        'base field' => 'parent_id',
        'field' => 'nid',
        'extra' => [
          ['field' => 'parent_type', 'value' => 'node'],
        ],
        'title' => t('Paragraphes'),
        'label' => t('paragraphes'),
      ],
    ];
  }
}
```

---

## Afficher le Rendu Complet d'un Paragraphe dans Views

```php
// Custom FieldHandler qui rend le paragraphe avec son view mode
// src/Plugin/views/field/ParagraphRender.php
namespace Drupal\mon_module\Plugin\views\field;

use Drupal\views\Plugin\views\field\FieldPluginBase;
use Drupal\views\ResultRow;

/**
 * @ViewsField("mon_module_paragraph_render")
 */
class ParagraphRender extends FieldPluginBase {

  public function query(): void {
    // Ajouter le champ ID du paragraphe à la query
    $this->ensureMyTable();
    $this->addAdditionalFields(['id', 'revision_id', 'type']);
  }

  public function preRender(array $values): void {
    // Batch loading de tous les paragraphes de la page
    $ids = [];
    foreach ($values as $row) {
      $id = $row->paragraphs_item_field_data_id ?? NULL;
      if ($id) $ids[$id] = $id;
    }

    if ($ids) {
      $this->paragraphs = \Drupal::entityTypeManager()
        ->getStorage('paragraph')
        ->loadMultiple($ids);
    }
  }

  public function render(ResultRow $values): array {
    $id = $values->paragraphs_item_field_data_id ?? NULL;
    $paragraph = $this->paragraphs[$id] ?? NULL;

    if (!$paragraph) return [];

    return \Drupal::entityTypeManager()
      ->getViewBuilder('paragraph')
      ->view($paragraph, 'default');
  }
}
```

---

## Template Views avec Paragraphs

```twig
{# views-view-fields--articles-paragraphs--page-1.html.twig #}
<article class="article-avec-paragraphs">
  <h2>{{ fields.title.content }}</h2>

  {# Accès au paragraphe rendu via le field handler custom #}
  {% if fields.paragraph_render is defined %}
    <div class="article__paragraphs">
      {{ fields.paragraph_render.content }}
    </div>
  {% endif %}

  {# Accès à un champ spécifique du paragraphe #}
  {% if fields.field_sous_titre is defined %}
    <p class="article__sous-titre">{{ fields.field_sous_titre.content }}</p>
  {% endif %}
</article>
```

---

## Performance — N+1 avec Views + Paragraphs

```
⚠️ RISQUE N+1 : chaque ligne de la View charge ses paragraphes séparément

Solution 1 — preRender batch loading (voir ParagraphRender ci-dessus)

Solution 2 — utiliser Views Aggregation + JOIN
  Views → Advanced → Use aggregation : YES
  → Évite les doublons si plusieurs paragraphes par nœud

Solution 3 — limiter la View
  → Pagination de 10-20 items max
  → Cache tag-based + contexte langue
```

---

## Filtrer par Contenu d'un Paragraphe dans une View de Nœuds

```yaml
# Filtre : nœuds dont le paragraphe type = "hero_image"
filters:
  type_1:
    id: type
    table: paragraphs_item_field_data
    field: type
    relationship: field_contenu_target_revision_id
    value:
      hero_image: hero_image
    plugin_id: string
```

---

## Commandes de Debug

```bash
# Voir quelles tables Paragraphs sont exposées à Views
drush php:eval "
\$data = \Drupal::service('views.views_data')->get('paragraphs_item_field_data');
var_dump(array_keys(\$data));
" | head -30

# Voir les relations disponibles
drush php:eval "
\$data = \Drupal::service('views.views_data')->get('node_field_data');
foreach (\$data as \$key => \$field) {
  if (isset(\$field['relationship'])) {
    echo \$key . ': ' . (\$field['relationship']['title'] ?? '?') . PHP_EOL;
  }
}
"
```
