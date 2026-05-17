---
name: drupal-content-modeling — taxonomy avancée
description: Taxonomy Drupal avancée - vocabulaires, termes hiérarchiques, API PHP, integration Views, taxonomy comme navigation vs classification, et performances.
---

# Taxonomy Avancée — Référence Complète

## Taxonomy : Classification vs Navigation

```
✅ Taxonomy pour :
  - Catégoriser du contenu (genres, thématiques, types)
  - Tags libres (free tagging)
  - Filtres dans Views (facettes, filtres exposés)
  - Hiérarchies de catégories (parent/enfant)
  - Relations M:N entre contenu et termes

❌ Taxonomy PAS pour :
  - Navigation principale du site → utiliser Menu links
  - Workflow éditorial → utiliser Content Moderation
  - Relations complex entre entités → utiliser Entity Reference
  - Données métier structurées → utiliser Custom Entities
```

---

## API PHP — Termes de Taxonomy

```php
use Drupal\taxonomy\Entity\Term;
use Drupal\taxonomy\Entity\Vocabulary;

// ── Charger un terme ────────────────────────────────────────────────────
$term = Term::load(42);
$name = $term->getName();
$description = $term->getDescription();
$vid = $term->bundle();           // ID du vocabulaire : 'tags', 'categories'
$parent_id = $term->parent->target_id;  // 0 = terme racine

// ── Charger plusieurs termes ─────────────────────────────────────────────
$terms = Term::loadMultiple([42, 43, 44]);

// ── Créer un terme ──────────────────────────────────────────────────────
$term = Term::create([
  'vid' => 'categories',
  'name' => 'JavaScript',
  'description' => [
    'value' => '<p>Ressources JavaScript.</p>',
    'format' => 'basic_html',
  ],
  'parent' => ['target_id' => 5],   // ID du terme parent (0 = racine)
  'weight' => 10,
  'field_couleur' => '#FF6B35',     // Champ custom
  'langcode' => 'fr',
]);
$term->save();

// ── Arbre taxonomique ────────────────────────────────────────────────────
$tree = \Drupal::entityTypeManager()
  ->getStorage('taxonomy_term')
  ->loadTree('categories', 0, NULL, TRUE);
// → array de Term entities ordonnés hiérarchiquement

// loadTree avec depth limit
$tree = \Drupal::entityTypeManager()
  ->getStorage('taxonomy_term')
  ->loadTree('categories', $parent_id, 2, TRUE);  // max 2 niveaux

// ── Terms d'un nœud ─────────────────────────────────────────────────────
$node = Node::load($nid);
$terms = $node->get('field_tags')->referencedEntities();

// ── Nœuds d'un terme ────────────────────────────────────────────────────
$nids = \Drupal::entityQuery('node')
  ->condition('field_tags', $term->id())
  ->condition('status', 1)
  ->accessCheck(TRUE)
  ->execute();
$nodes = Node::loadMultiple($nids);

// ── Chercher un terme par nom ────────────────────────────────────────────
$terms = \Drupal::entityTypeManager()
  ->getStorage('taxonomy_term')
  ->loadByProperties(['name' => 'JavaScript', 'vid' => 'categories']);
$term = reset($terms) ?: NULL;

// ── Créer ou charger un terme (upsert) ──────────────────────────────────
function get_or_create_term(string $name, string $vid): Term {
  $terms = \Drupal::entityTypeManager()
    ->getStorage('taxonomy_term')
    ->loadByProperties(['name' => $name, 'vid' => $vid]);

  if ($terms) {
    return reset($terms);
  }

  $term = Term::create(['vid' => $vid, 'name' => $name]);
  $term->save();
  return $term;
}
```

---

## Taxonomy Hiérarchique

```php
// Obtenir les parents d'un terme
$parents = \Drupal::entityTypeManager()
  ->getStorage('taxonomy_term')
  ->loadParents($term->id());
// → array de Term entities (directs parents seulement)

// Obtenir tous les ancêtres (breadcrumb)
$all_parents = \Drupal::entityTypeManager()
  ->getStorage('taxonomy_term')
  ->loadAllParents($term->id());
// → array ordonné du terme lui-même jusqu'à la racine

// Obtenir les enfants d'un terme
$children = \Drupal::entityTypeManager()
  ->getStorage('taxonomy_term')
  ->loadChildren($term->id());
// → array de Term entities (enfants directs)

// Vérifier si un terme est ancêtre d'un autre
function is_ancestor_of(int $ancestor_id, int $term_id): bool {
  $parents = \Drupal::entityTypeManager()
    ->getStorage('taxonomy_term')
    ->loadAllParents($term_id);
  return isset($parents[$ancestor_id]);
}
```

---

## Afficher la Hiérarchie dans Twig

```twig
{# Afficher le fil d'Ariane taxonomy #}
{# Dans un preprocess, calculer $variables['term_parents'] #}

{% if term_parents %}
  <nav class="taxonomy-breadcrumb" aria-label="{{ 'Navigation taxonomique'|t }}">
    {% for parent in term_parents|reverse %}
      <a href="{{ parent.url }}" class="taxonomy-breadcrumb__link">{{ parent.name }}</a>
      <span class="taxonomy-breadcrumb__sep" aria-hidden="true">›</span>
    {% endfor %}
    <span class="taxonomy-breadcrumb__current">{{ term.name.value }}</span>
  </nav>
{% endif %}

{# Afficher les tags d'un article #}
{% if content.field_tags %}
  <ul class="article-tags">
    {% for item in content.field_tags %}
      <li class="article-tags__item">{{ item }}</li>
    {% endfor %}
  </ul>
{% endif %}
```

---

## Views avec Taxonomy

```
# Configuration Views recommandée pour une page taxonomique :

1. Source de données : Taxonomy Term
2. Contextual filter :  Taxonomy term: Term ID (from URL)
   → Valider avec : "Taxonomy term" (valide que l'argument est un TID)
   → Quand absent : "Page not found" (404)

# Pour lister les nœuds d'un terme :

1. Source : Content
2. Relationship : Content: field_tags (relationship to taxonomy_term)
   → ou Filtrer sur field_tags directement
3. Contextual filter : Taxonomy term: Term ID from URL
   → filtrer via la relation

# Vue des sous-termes d'un vocabulaire :
1. Source : Taxonomy Term
2. Filter : Term: Vocabulary (= 'categories')
3. Filter : Term: Parent term (contextual → ID du terme parent)
```

---

## Performance Taxonomy

```php
// ❌ Charger l'arbre taxonomy sans limit → problème si > 500 termes
$tree = \Drupal::entityTypeManager()
  ->getStorage('taxonomy_term')
  ->loadTree('categories', 0, NULL, TRUE);

// ✅ Limiter la profondeur
$tree = \Drupal::entityTypeManager()
  ->getStorage('taxonomy_term')
  ->loadTree('categories', 0, 2, FALSE);  // max 2 niveaux, sans charger les entités

// ✅ EntityQuery paginée pour les vocabulaires larges
$query = \Drupal::entityQuery('taxonomy_term')
  ->condition('vid', 'tags')
  ->sort('name')
  ->range(0, 50)
  ->accessCheck(FALSE);
$tids = $query->execute();

// ✅ Cache les arbres taxonomy
$cid = 'mon_module:taxonomy_tree:categories';
$cached = \Drupal::cache()->get($cid);
if ($cached) {
  $tree = $cached->data;
} else {
  $tree = \Drupal::entityTypeManager()
    ->getStorage('taxonomy_term')
    ->loadTree('categories', 0, 3, TRUE);
  \Drupal::cache()->set($cid, $tree, \Drupal\Core\Cache\Cache::PERMANENT, ['taxonomy_term_list:categories']);
}
```
