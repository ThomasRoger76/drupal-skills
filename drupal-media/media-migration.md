---
name: drupal-media — migration
description: Migrer des fichiers et médias vers Drupal avec la Migrate API - migration D7 files, CSV avec chemins locaux, création de File + Media entities, et gestion des fichiers manquants.
---

# Migration de Médias — Référence Complète

## Stratégie de Migration Média

```
Source (D7/CSV/dossier)
  → Créer entité File (gérer le fichier physique)
    → Créer entité Media (enveloppe avec métadonnées)
      → Référencer depuis Node/Paragraph

Ordre obligatoire :
1. Migrer les files (d7_file ou source CSV)
2. Migrer les media entities (d7_file_entity ou source custom)
3. Migrer les nodes qui référencent les media
```

---

## Migrer les Fichiers D7 → File Entities D10

```yaml
# config/install/migrate_plus.migration.d7_file_to_d10.yml
id: d7_file_to_d10
label: 'D7 Files → D10 File Entities'
migration_group: d7_migration

source:
  plugin: d7_file
  scheme: public
  # Connexion D7 définie dans settings.php :
  # $databases['migrate']['default'] = [...]
  key: migrate

destination:
  plugin: 'entity:file'

process:
  filename: filename
  uri:
    plugin: file_copy
    source:
      - filepath      # Chemin physique D7
      - uri           # URI D10 cible (public://...)
    file_exists: rename
  status: status
  created: timestamp
  changed: timestamp
  uid: uid
  filemime: filemime
  filesize: filesize

migration_dependencies:
  required: []
```

---

## Migrer D7 File Entities → D10 Media

```yaml
# config/install/migrate_plus.migration.d7_media_image.yml
id: d7_media_image
label: 'D7 Images → D10 Media (image bundle)'
migration_group: d7_migration

source:
  plugin: d7_file_entity
  type: image   # Type de file entity D7

destination:
  plugin: 'entity:media'
  default_bundle: image

process:
  name: filename
  status: status
  created: timestamp
  changed: timestamp
  uid: uid
  langcode:
    plugin: default_value
    default_value: fr
  'field_media_image/target_id':
    plugin: migration_lookup
    migration: d7_file_to_d10
    source: fid
    no_stub: true
  'field_media_image/alt':
    plugin: default_value
    default_value: ''
  'field_media_image/title': filename

migration_dependencies:
  required:
    - d7_file_to_d10
```

---

## Migrer depuis un Dossier Local (CSV + fichiers)

```yaml
# config/install/migrate_plus.migration.import_images_csv.yml
id: import_images_csv
label: 'Import images depuis CSV + dossier local'
migration_group: custom_import

source:
  plugin: csv
  path: 'public://import/images.csv'
  # CSV format : id,filename,alt_text,legende
  delimiter: ','
  enclosure: '"'
  header_row_count: 1
  ids:
    - id
  fields:
    - name: id
    - name: filename
    - name: alt_text
    - name: legende

destination:
  plugin: 'entity:media'
  default_bundle: image

process:
  name: legende
  status:
    plugin: default_value
    default_value: 1
  langcode:
    plugin: default_value
    default_value: fr

  # Copier le fichier physique + créer l'entité File
  'field_media_image/target_id':
    plugin: file_import
    source: filename
    destination: 'public://medias/[date:custom:Y]-[date:custom:m]/'
    # Le fichier source doit être accessible depuis Drupal
    # (ex: dans public://import/source/)
    file_exists: use_existing
    id_only: true

  'field_media_image/alt': alt_text
  'field_media_image/title': legende

migration_dependencies:
  required: []
```

---

## Plugin Source Custom — Fichiers sur Système Local

```php
<?php
// src/Plugin/migrate/source/LocalImageFiles.php
namespace Drupal\mon_module\Plugin\migrate\source;

use Drupal\migrate\Plugin\migrate\source\SourcePluginBase;
use Drupal\migrate\Row;

/**
 * Source plugin pour les images d'un dossier local.
 *
 * @MigrateSource(
 *   id = "local_image_files"
 * )
 */
class LocalImageFiles extends SourcePluginBase {

  public function getIds(): array {
    return ['filepath' => ['type' => 'string']];
  }

  public function fields(): array {
    return [
      'filepath' => $this->t('Chemin absolu du fichier'),
      'filename' => $this->t('Nom du fichier'),
      'extension' => $this->t('Extension'),
      'filesize' => $this->t('Taille en octets'),
    ];
  }

  public function __toString(): string {
    return 'local_image_files:' . $this->configuration['directory'];
  }

  protected function initializeIterator(): \Iterator {
    $directory = $this->configuration['directory'];
    $extensions = $this->configuration['extensions'] ?? ['jpg', 'jpeg', 'png', 'webp'];

    $files = [];
    foreach (new \DirectoryIterator($directory) as $file) {
      if ($file->isDot() || !$file->isFile()) { continue; }
      if (!in_array(strtolower($file->getExtension()), $extensions)) { continue; }

      $files[] = [
        'filepath' => $file->getPathname(),
        'filename' => $file->getFilename(),
        'extension' => $file->getExtension(),
        'filesize' => $file->getSize(),
      ];
    }

    return new \ArrayIterator($files);
  }
}
```

---

## Gérer les Fichiers Manquants

```php
// Vérifier les media avec fichiers manquants après migration
drush php:eval "
\$mids = \Drupal::entityQuery('media')
  ->condition('bundle', 'image')
  ->accessCheck(FALSE)
  ->execute();

\$manquants = 0;
foreach (array_chunk(\$mids, 50) as \$chunk) {
  \$medias = \Drupal::entityTypeManager()->getStorage('media')->loadMultiple(\$chunk);
  foreach (\$medias as \$media) {
    \$file = \$media->get('field_media_image')->entity;
    if (!\$file || !\$file->getSize()) {
      echo 'Media ' . \$media->id() . ': fichier manquant' . PHP_EOL;
      \$manquants++;
    }
  }
}
echo \$manquants . ' media(s) avec fichier manquant';
"
```

---

## Commandes Drush Migration Médias

```bash
# Lancer la migration des fichiers D7
drush migrate:import d7_file_to_d10 --feedback=500

# Lancer la migration des media
drush migrate:import d7_media_image --feedback=100

# Vérifier le statut
drush migrate:status --group=d7_migration

# Rollback si problème
drush migrate:rollback d7_media_image
drush migrate:rollback d7_file_to_d10

# Debug — voir les erreurs
drush migrate:import d7_media_image --feedback=1 2>&1 | grep -E "ERROR|WARNING|Processed"
```
