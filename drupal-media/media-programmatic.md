---
name: drupal-media — programmation
description: Accéder et manipuler les Media entities Drupal depuis PHP - Media::load, Media::create, file_url_generator, accès aux champs, et intégration JSON:API.
---

# Media Programmatique — API PHP Complète

## Charger un Media

```php
use Drupal\media\Entity\Media;

// Charger par ID
$media = Media::load($mid);

// Charger plusieurs
$medias = Media::loadMultiple([$mid1, $mid2, $mid3]);

// Charger depuis un champ Entity Reference
$node = \Drupal\node\Entity\Node::load($nid);
$media = $node->get('field_image')->entity;   // Un seul
$medias = $node->get('field_images')->referencedEntities();  // Plusieurs

// Requête EntityQuery
$mids = \Drupal::entityQuery('media')
  ->condition('bundle', 'image')
  ->condition('status', 1)
  ->accessCheck(TRUE)
  ->execute();
$images = Media::loadMultiple($mids);
```

---

## Créer un Media depuis un Fichier

```php
use Drupal\Core\File\FileExists;
use Drupal\media\Entity\Media;

// ✅ D10.3+ — file.repository->writeData() crée le File managé en une étape
// (gère le déplacement physique + l'entité File). N'utilise PAS File::create()
// + setFileUri() manuel : ça ne déplace pas le fichier physique.
$data = file_get_contents('/chemin/source/mon-image.jpg');
$file = \Drupal::service('file.repository')->writeData(
  $data,
  'public://images/mon-image.jpg',
  FileExists::Rename,   // ← enum D10.3+, remplace FILE_EXISTS_RENAME / FileSystemInterface::EXISTS_*
);

// OU depuis une URL distante — system_retrieve_file() est DÉPRÉCIÉ (D10.3+).
// Utiliser le client HTTP + file.repository :
$response = \Drupal::httpClient()->get('https://example.com/image.jpg');
$file = \Drupal::service('file.repository')->writeData(
  (string) $response->getBody(),
  'public://images/image.jpg',
  FileExists::Rename,
);

// Étape 2 — Créer l'entité Media
$media = Media::create([
  'bundle' => 'image',
  'name' => 'Mon image',
  'status' => 1,
  'uid' => \Drupal::currentUser()->id(),
  'field_media_image' => [
    'target_id' => $file->id(),
    'alt' => 'Description de l\'image',
    'title' => 'Titre de l\'image',
  ],
  // Champs custom
  'field_legende' => 'Légende de l\'image',
  'langcode' => 'fr',
]);
$media->save();

echo 'Media créé : ' . $media->id();
```

---

## Obtenir l'URL d'un Fichier Media

```php
// ✅ D10+ — file_url_generator service (file_create_url() supprimé)
$file_url_generator = \Drupal::service('file_url_generator');

// URL absolue
$media = Media::load($mid);
$file = $media->get('field_media_image')->entity;
$url_absolute = $file_url_generator->generateAbsoluteString($file->getFileUri());
// → 'https://mon-site.com/sites/default/files/images/mon-image.jpg'

// URL relative
$url_relative = $file_url_generator->generateString($file->getFileUri());
// → '/sites/default/files/images/mon-image.jpg'

// URI du fichier (scheme://)
$uri = $file->getFileUri();
// → 'public://images/mon-image.jpg'

// Avec image style
$image_style = \Drupal::entityTypeManager()->getStorage('image_style')->load('large');
$url_styled = $image_style->buildUrl($file->getFileUri());
// → 'https://mon-site.com/sites/default/files/styles/large/public/images/mon-image.jpg'
```

---

## Accéder aux Métadonnées d'une Image

```php
$media = Media::load($mid);
$image_field = $media->get('field_media_image');

// Métadonnées du champ image
$alt = $image_field->alt;
$title = $image_field->title;
$width = $image_field->width;
$height = $image_field->height;
$file_id = $image_field->target_id;

// Entité File associée
$file = $image_field->entity;
$file_uri = $file->getFileUri();
$file_size = $file->getSize();
$mime_type = $file->getMimeType();
$file_name = $file->getFilename();
```

---

## Preprocess — Media dans Twig

```php
// hook_preprocess_node — exposer les données d'un Media au template
function mon_theme_preprocess_node(array &$variables): void {
  $node = $variables['node'];

  if (!$node->hasField('field_image') || $node->get('field_image')->isEmpty()) {
    return;
  }

  $media = $node->get('field_image')->entity;
  if (!$media instanceof \Drupal\media\Entity\MediaInterface) {
    return;
  }

  // Charger la traduction dans le contexte courant
  $media = \Drupal::service('entity.repository')->getTranslationFromContext($media);

  $file = $media->get('field_media_image')->entity;
  if (!$file) {
    return;
  }

  $file_url_generator = \Drupal::service('file_url_generator');

  $variables['image_data'] = [
    'url' => $file_url_generator->generateAbsoluteString($file->getFileUri()),
    'alt' => $media->get('field_media_image')->alt,
    'title' => $media->get('field_media_image')->title,
    'width' => $media->get('field_media_image')->width,
    'height' => $media->get('field_media_image')->height,
    'mime' => $file->getMimeType(),
  ];

  // Cache context langue (si Media traduit)
  $variables['#cache']['contexts'][] = 'languages:language_content';
  $variables['#cache']['tags'] = array_merge(
    $variables['#cache']['tags'] ?? [],
    $media->getCacheTags()
  );
}
```

---

## JSON:API avec Media

```bash
# Inclure les données de media dans une réponse JSON:API
GET /jsonapi/node/article?include=field_image,field_image.field_media_image

# Réponse inclut :
# data.relationships.field_image → Media entity
# included[] → Media entity + File entity
# → URL du fichier, alt, width, height dans field_media_image

# Accéder à l'URL de l'image depuis le frontend
const imageUrl = included
  .find(r => r.id === article.relationships.field_image.data.id)
  ?.relationships?.field_media_image?.data;
```

---

## Nettoyer les Médias Orphelins

> **Piège :** `file.usage->listUsage($file)` retourne l'usage du **File** — il
> inclut le Media qui le porte, donc il n'est jamais vide pour un Media valide.
> Ça ne dit PAS si le Media lui-même est référencé par un nœud. Pour tracker
> les références **vers une entité Media**, utiliser le contrib `drupal/entity_usage`.

```php
// ✅ Avec drupal/entity_usage (contrib) — tracke les références entrantes
// vers n'importe quelle entité, y compris Media.
function trouver_medias_orphelins(): array {
  $mids = \Drupal::entityQuery('media')
    ->condition('bundle', 'image')
    ->condition('status', 1)
    ->accessCheck(FALSE)
    ->execute();

  $entity_usage = \Drupal::service('entity_usage.usage');
  $storage = \Drupal::entityTypeManager()->getStorage('media');

  $orphelins = [];
  foreach (array_chunk($mids, 100) as $chunk) {
    foreach ($storage->loadMultiple($chunk) as $media) {
      // Sources qui référencent ce Media (nodes, paragraphs, blocks...)
      $usage = $entity_usage->listSources($media);
      if (empty($usage)) {
        $orphelins[] = $media->id();
      }
    }
  }

  return $orphelins;
}

// Supprimer les orphelins (en batch)
$orphelins = trouver_medias_orphelins();
$storage = \Drupal::entityTypeManager()->getStorage('media');
$storage->delete($storage->loadMultiple($orphelins));
```
