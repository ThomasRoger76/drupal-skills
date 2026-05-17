---
name: drupal-content-modeling — field types
description: Guide de sélection des types de champs Drupal - quand utiliser String vs Text, Date vs DateTime vs Daterange, Image vs File vs Media, Link, Boolean, Entity Reference, et les champs custom.
---

# Types de Champs Drupal — Guide de Sélection

## Tableau de Décision Rapide

| Besoin | Type de champ | Module |
|--------|---------------|--------|
| Texte court (titre, sous-titre, étiquette) | **String** (`string`) | Core |
| Texte long sans HTML (description, résumé) | **Plain text long** (`string_long`) | Core |
| Texte riche WYSIWYG (corps d'article) | **Text with summary** (`text_with_summary`) | Core |
| Texte formaté sans résumé | **Text** (`text_long`) | Core |
| Nombre entier (âge, quantité, score) | **Integer** (`integer`) | Core |
| Prix, pourcentage (décimal) | **Decimal** (`decimal`) | Core |
| Mesure flottante (poids, température) | **Float** (`float`) | Core |
| Oui/Non, actif/inactif | **Boolean** (`boolean`) | Core |
| Date seule (anniversaire, date de naissance) | **Date** (`datetime` avec time=false) | Core |
| Date + heure précise (événement) | **DateTime** (`datetime`) | Core |
| Plage de dates (événement avec fin) | **Daterange** (`daterange`) | Core |
| Couleur hexadécimale | **Color** (`color_field_type`) | `drupal/color_field` |
| Email | **Email** (`email`) | Core |
| Téléphone | **Telephone** (`telephone`) | Core |
| Lien (URL + texte cliquable) | **Link** (`link`) | Core |
| Fichier uploadé (PDF, ZIP...) | **File** (`file`) | Core |
| Image avec alt + title + crop | **Image** (`image`) | Core |
| Média réutilisable (image, vidéo, document) | **Media entity reference** | Core (Media Library) |
| Référence à une entité Drupal | **Entity reference** (`entity_reference`) | Core |
| Référence à un paragraphe | **Entity reference revisions** | `drupal/paragraphs` |
| Géolocalisation (lat/lon) | **Geofield** | `drupal/geofield` |
| Adresse structurée (pays, ville, code postal) | **Address** | `drupal/address` |
| Liste de valeurs fixes (select) | **List** (`list_string`, `list_integer`) | Core |

---

## Texte — Choisir le Bon Type

```
Texte court (< 255 chars, pas de HTML)
  → field_type: string
  → Widget: textfield
  → Exemples: sous-titre, tag, code produit

Texte long (> 255 chars, pas de HTML)
  → field_type: string_long
  → Widget: textarea
  → Exemples: meta description, résumé textuel

Texte formaté sans résumé (HTML autorisé)
  → field_type: text_long
  → Widget: text_textarea_with_summary sans summary
  → Exemples: description de produit avec <ul>, <strong>

Texte avec résumé (corps d'article complet)
  → field_type: text_with_summary
  → Widget: text_textarea_with_summary
  → Exemples: body d'un article, contenu d'une page
  → Accès: $node->body->value (HTML), $node->body->summary (résumé)
```

**Règle de base :**
- Si l'éditeur n'a pas besoin de mise en forme → `string` ou `string_long`
- Si l'éditeur a besoin de `<strong>`, listes, liens → `text_long` ou `text_with_summary`
- Si tu veux stocker du HTML généré par code → `text_long` avec `format: full_html`

---

## Date — Choisir le Bon Type

```yaml
# Date seule (pas d'heure, pas de timezone)
# Stocké en DB : YYYY-MM-DD
field_type: datetime
datetime_type: date       # ← clé importante

# DateTime (date + heure, avec timezone)
# Stocké en DB : YYYY-MM-DDTHH:MM:SS (UTC)
field_type: datetime
datetime_type: datetime

# Plage de dates (start + end dans un seul champ)
# Nécessite drupal/datetime_range (core depuis D8.5)
field_type: daterange
datetime_type: datetime   # ou date pour dates seules
```

```php
// Accéder à la valeur d'une date
$date = $node->get('field_date_evenement')->date;
// → DrupalDateTime object

$timestamp = $node->get('field_date_evenement')->value;
// → string : '2026-05-16T14:30:00' (UTC)

// Formater une date
$formatter = \Drupal::service('date.formatter');
$date_str = $formatter->format($timestamp_unix, 'custom', 'd/m/Y H:i');

// Plage de dates
$range = $node->get('field_periode');
$debut = $range->value;
$fin = $range->end_value;
```

---

## Image vs File vs Media — Choisir

```
File field :
  - Stocke un fichier et son URI
  - Pas de métadonnées (alt, title, width, height)
  - Pas réutilisable entre entités
  → Quand : uploads simples (PDF, documents) sans galerie

Image field :
  - Hérite de File + alt obligatoire + dimensions
  - Pas réutilisable entre entités
  - Image styles disponibles
  → Quand : image unique sur une entité, contexte simple

Media entity reference :
  - Entité à part entière avec ses propres champs
  - RÉUTILISABLE entre nœuds (un fichier, plusieurs usages)
  - Media Library native (D9+)
  - Supporte images, vidéos, documents, tweets, etc.
  → Recommandé pour tout site avec une médiathèque
  → Quand : sites avec gestion des médias, galeries, vidéos
```

```php
// Accéder à un media entity reference
$media_item = $node->get('field_image')->entity;
// → MediaInterface

if ($media_item && $media_item->bundle() === 'image') {
  $file = $media_item->get('field_media_image')->entity;
  $uri = $file->getFileUri();
  $url = \Drupal::service('file_url_generator')->generateAbsoluteString($uri);

  $alt = $media_item->get('field_media_image')->alt;
  $title = $media_item->get('field_media_image')->title;
}
```

---

## Entity Reference — Configuration

```yaml
# Référence vers un Node (article, page...)
field_type: entity_reference
settings:
  handler: default:node
  handler_settings:
    target_bundles:
      article: article
    sort:
      field: title
      direction: ASC
    auto_create: false

# Référence vers n'importe quel type d'entité
settings:
  handler: default
  # → tous les types d'entités disponibles

# Référence Taxonomy
field_type: entity_reference
settings:
  handler: default:taxonomy_term
  handler_settings:
    target_bundles:
      tags: tags
      categories: categories
```

```php
// Accéder à une entity reference
$referenced_nodes = $node->get('field_articles_lies')->referencedEntities();
foreach ($referenced_nodes as $referenced_node) {
  echo $referenced_node->getTitle();
}

// Vérifier si un champ reference est vide
if (!$node->get('field_categorie')->isEmpty()) {
  $term = $node->get('field_categorie')->entity;
}
```

---

## List — Valeurs Fixes (Select / Radio)

```yaml
# Liste de valeurs de type string
field_type: list_string
settings:
  allowed_values:
    - value: pending
      label: 'En attente'
    - value: confirmed
      label: 'Confirmé'
    - value: cancelled
      label: 'Annulé'

# Liste d'entiers
field_type: list_integer
settings:
  allowed_values:
    - value: 1
      label: 'Priorité basse'
    - value: 2
      label: 'Priorité normale'
    - value: 3
      label: 'Priorité haute'
```

```php
// Accéder à la valeur
$statut = $node->get('field_statut')->value;          // 'pending'

// Obtenir le label
$allowed_values = $node->getFieldDefinition('field_statut')
  ->getFieldStorageDefinition()
  ->getSetting('allowed_values');
$label = $allowed_values[$statut]['label'] ?? $statut;

// Plus simplement via FieldDefinition
$options = \Drupal\options\Plugin\Field\FieldType\ListTextItem::processAllowedValues($definition);
```

---

## Cardinality — Champs Multi-Valeurs

```php
// Accéder à un champ multi-valeur
foreach ($node->get('field_tags') as $delta => $item) {
  $term = $item->entity;
  echo $term->getName();
}

// Accéder à la première valeur seulement
$first_tag = $node->get('field_tags')->first()->entity ?? NULL;

// Compter les valeurs
$count = $node->get('field_images')->count();

// Ajouter une valeur
$node->get('field_tags')->appendItem(['target_id' => $term_id]);

// Remplacer toutes les valeurs
$node->set('field_tags', [
  ['target_id' => 15],
  ['target_id' => 23],
]);
```
