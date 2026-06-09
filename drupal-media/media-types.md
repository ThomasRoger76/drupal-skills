---
name: drupal-media — media types
description: Configurer les Media types Drupal (image, video, document, remote_video, audio), leurs sources, champs, et créer un Media source plugin custom.
---

# Media Types — Configuration Complète

## Media Types Natifs Drupal

```
/admin/structure/media/add

Media types disponibles par défaut :
├── Image        → source: image, champ: field_media_image
├── Remote video → source: oembed:video (YouTube, Vimeo...)
├── Document     → source: file, champ: field_media_document
├── Video        → source: video_file, champ: field_media_video_file
└── Audio        → source: audio_file, champ: field_media_audio_file
```

---

## Configurer un Media Type via YAML

```yaml
# config/install/media_type.media_type.image.yml
langcode: fr
status: true
id: image
label: 'Image'
description: 'Images réutilisables dans la bibliothèque de médias.'
source: image
source_configuration:
  source_field: field_media_image
queue_thumbnail_downloads: false
new_revision: false
```

```yaml
# Champ source de l'image
# config/install/field.field.media.image.field_media_image.yml
langcode: fr
status: true
id: media.image.field_media_image
field_name: field_media_image
entity_type: media
bundle: image
label: 'Image'
required: true
settings:
  file_extensions: 'png gif jpg jpeg webp svg'
  max_filesize: '10 MB'
  max_resolution: ''
  min_resolution: ''
  alt_field: true
  alt_field_required: true    # ← OBLIGATOIRE pour l'accessibilité
  title_field: false
  title_field_required: false
```

---

## Remote Video (oEmbed)

```yaml
# Media type Remote Video
source: oembed:video
source_configuration:
  source_field: field_media_oembed_video
  thumbnails_directory: 'public://oembed_thumbnails'
  providers:
    - YouTube
    - Vimeo
```

```php
// Accéder à l'URL d'une vidéo Remote Video
$media = \Drupal\media\Entity\Media::load($mid);
$source = $media->getSource();
$video_url = $source->getSourceFieldValue($media);
// → 'https://www.youtube.com/watch?v=...'

// Afficher via Twig
// {{ content.field_video }} → embed oEmbed automatique
```

**Whitelist des providers oEmbed :**
```yaml
# config/install/media.settings.yml
iframe_domain: ''    # Laisser vide pour le domaine courant
oembed_providers:
  - https://oembed.com/providers.json    # Inclut YouTube, Vimeo, etc.
```

---

## Champs Custom sur un Media Type

```yaml
# Ajouter un champ "Légende" sur le type Image
# config/install/field.storage.media.field_legende.yml
langcode: fr
status: true
id: media.field_legende
field_name: field_legende
entity_type: media
type: string
cardinality: 1

# config/install/field.field.media.image.field_legende.yml
langcode: fr
status: true
id: media.image.field_legende
field_name: field_legende
entity_type: media
bundle: image
label: 'Légende'
required: false
translatable: true    # ← Traduit par langue si site multilingue
```

---

## Media Source Plugin Custom

```php
<?php
// src/Plugin/media/Source/MonApiMedia.php
namespace Drupal\mon_module\Plugin\media\Source;

use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\media\Attribute\MediaSource;
use Drupal\media\MediaInterface;
use Drupal\media\MediaSourceBase;

/**
 * Source Media pour une API externe.
 *
 * D11 : attribut PHP #[MediaSource] (l'annotation @MediaSource est dépréciée
 * et supprimée en D12). Disponible depuis D10.2.
 */
#[MediaSource(
  id: 'mon_api_media',
  label: new TranslatableMarkup('Mon API'),
  description: new TranslatableMarkup('Médias hébergés sur notre API externe.'),
  allowed_field_types: ['string'],
  default_thumbnail_filename: 'generic.png',
)]
class MonApiMedia extends MediaSourceBase {

  /**
   * Retourner la valeur du champ source (ex: ID ou URL du média).
   */
  public function getSourceFieldValue(MediaInterface $media): ?string {
    $source_field = $this->configuration['source_field'];
    return $media->get($source_field)->value;
  }

  /**
   * Métadonnées disponibles pour ce type de source.
   */
  public function getMetadataAttributes(): array {
    return [
      'title' => $this->t('Titre du média'),
      'thumbnail_uri' => $this->t('URI de la miniature'),
      'duration' => $this->t('Durée (secondes)'),
    ];
  }

  /**
   * Obtenir une métadonnée spécifique.
   */
  public function getMetadata(MediaInterface $media, $attribute_name) {
    $source_value = $this->getSourceFieldValue($media);

    return match($attribute_name) {
      'title' => $this->fetchApiTitle($source_value),
      'thumbnail_uri' => $this->fetchApiThumbnail($source_value),
      'default_name' => 'Media #' . $media->id(),
      default => parent::getMetadata($media, $attribute_name),
    };
  }

  private function fetchApiTitle(string $id): string {
    // Appel API externe pour récupérer le titre
    return 'Titre du média ' . $id;
  }
}
```

---

## Formatters de Media

```
/admin/structure/media/manage/image/display

Formatters disponibles pour un Media type Image :
├── "Image" → rendu HTML <img>
├── "Responsive image" → <picture> avec srcset
├── "URL" → URL brute du fichier
└── "Thumbnail" → rendu avec image style

Configuration :
  Image style → thumbnail, medium, large, custom
  Link image to → content, file
  Alt text → from field_media_image.alt
```

```yaml
# config/install/core.entity_view_display.media.image.default.yml
display:
  field_media_image:
    type: image
    settings:
      image_style: large
      image_link: ''
    weight: 0
```
