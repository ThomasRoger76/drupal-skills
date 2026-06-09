---
name: drupal-media — media library
description: Configurer et utiliser la Media Library Drupal (core D9+) - widget de sélection, bulk upload, Focal Point, Image Styles, et responsive images pour les médias.
---

# Media Library — Configuration & Usage

## Activation

```bash
# Media Library est dans le core Drupal 9+ mais nécessite d'être activé
drush en media media_library -y

# Module Focal Point (recadrage centré)
composer require drupal/focal_point
drush en focal_point -y
```

---

## Widget Media Library dans un Formulaire

```
/admin/structure/types/manage/article/fields

→ Ajouter un champ → Entity Reference → Media
→ Widget : Media Library (remplace "Entity Browser" ou "Autocomplete")

Configuration du widget :
  ├── Media types autorisés : image, video, document
  ├── Cardinality : 1 (image principale) ou illimité (galerie)
  └── Extensions d'image autorisées : png jpg jpeg webp
```

---

## Focal Point — Recadrage Intelligent

```
Installation activée → le champ Image affiche un crosshair sur l'aperçu
→ L'éditeur clique pour définir le point focal
→ Tous les Image Styles recadrent en gardant ce point visible

Configuration Image Style avec Focal Point :
  /admin/config/media/image-styles → Add Style → Add Effect
  → "Focal Point Scale And Crop" (au lieu de "Scale and Crop" standard)
  → Width: 800, Height: 450 → Focal point respecté
```

---

## Image Styles — Configuration

```bash
# Lister les image styles disponibles
drush php:eval "
\$styles = \Drupal::entityTypeManager()->getStorage('image_style')->loadMultiple();
foreach (\$styles as \$style) {
  echo \$style->id() . ': ' . \$style->label() . PHP_EOL;
}
"

# Générer un dérivé d'image style pour un fichier
drush php:eval "
\$style = \Drupal::entityTypeManager()->getStorage('image_style')->load('large');
\$uri = 'public://2026-05/mon-image.jpg';
echo \$style->buildUrl(\$uri) . PHP_EOL;
"

# Vider le cache des dérivés d'image (après modification d'un style)
# Flush d'un style précis (régénère ses dérivés au prochain accès)
drush php:eval "\Drupal::entityTypeManager()->getStorage('image_style')->load('large')->flush();"
# OU tous les styles
drush image:flush --all
```

---

## Responsive Images — Configuration Complète

```yaml
# 1. Breakpoints — web/themes/custom/mon_theme/mon_theme.breakpoints.yml
mon_theme.mobile:
  label: Mobile
  mediaQuery: 'all and (max-width: 767px)'
  weight: 0
  multipliers: ['1x', '2x']

mon_theme.tablet:
  label: Tablet
  mediaQuery: 'all and (min-width: 768px) and (max-width: 1023px)'
  weight: 1
  multipliers: ['1x', '2x']

mon_theme.desktop:
  label: Desktop
  mediaQuery: 'all and (min-width: 1024px)'
  weight: 2
  multipliers: ['1x', '2x']
```

```bash
# 2. Créer les Image Styles pour chaque breakpoint
# /admin/config/media/image-styles
# → article_mobile (400px), article_tablet (768px), article_desktop (1200px)

# 3. Créer un Responsive Image Style
# /admin/config/media/responsive-image-style → Add
# → Breakpoint group : mon_theme
# → Pour chaque breakpoint : sélectionner l'Image Style correspondant

# 4. Configurer le formatter dans le display
# /admin/structure/types/manage/article/display
# → field_image → Formatter : Responsive Image
# → Responsive Image Style : article_hero
```

---

## Bulk Upload — Upload Multiple Médias

```
/admin/content/media → Add Media → Image

Upload multiple fichiers :
  → Glisser-déposer plusieurs images dans la Media Library
  → Chaque image crée automatiquement une entité Media
  → Alt text requis pour chaque image

Via Drush (import en masse depuis un dossier) :
```

```php
// Script de bulk upload depuis un dossier
use Drupal\Core\File\FileExists;
use Drupal\media\Entity\Media;

function bulk_import_images(string $directory): void {
  $files = glob($directory . '/*.jpg');

  foreach ($files as $file_path) {
    $filename = basename($file_path);
    $uri = 'public://imports/' . $filename;

    // Copier le fichier ET créer l'entité File managée en une étape.
    // file.repository->copy() retourne directement le File (déplacement
    // physique + entité). FileExists::Replace remplace le déprécié
    // FileSystemInterface::EXISTS_REPLACE (D10.3+).
    $file = \Drupal::service('file.repository')->copy(
      $file_path,
      $uri,
      FileExists::Replace,
    );

    // Créer l'entité Media
    $media = Media::create([
      'bundle' => 'image',
      'name' => pathinfo($filename, PATHINFO_FILENAME),
      'status' => 1,
      'field_media_image' => [
        'target_id' => $file->id(),
        'alt' => pathinfo($filename, PATHINFO_FILENAME),
      ],
    ]);
    $media->save();
    
    echo "Importé : $filename → Media ID " . $media->id() . PHP_EOL;
  }
}
```

---

## Configurer la Media Library dans un Formulaire Custom

```php
// Dans un formulaire custom — champ Media Library
$form['image'] = [
  '#type' => 'media_library',
  '#allowed_bundles' => ['image'],
  '#title' => $this->t('Image principale'),
  '#default_value' => NULL,  // ou Media ID existant
  '#description' => $this->t('Sélectionner ou uploader une image.'),
  '#cardinality' => 1,
];

// Récupérer la valeur soumise
$media_id = $form_state->getValue('image');
if ($media_id) {
  $media = \Drupal\media\Entity\Media::load($media_id);
}
```

---

## Troubleshooting

```bash
# Media non visibles dans la Library → vérifier le statut
drush php:eval "
\$mids = \Drupal::entityQuery('media')->condition('status', 0)->accessCheck(FALSE)->execute();
echo count(\$mids) . ' médias non publiés' . PHP_EOL;
"

# Régénérer les dérivés d'image après changement de style
drush image:flush --all

# Vérifier les permissions (les utilisateurs voient-ils la Media Library ?)
# → /admin/people/permissions → Media Library → "Use Media overview"

# Files orphelines (pas dans une entité Media)
drush php:eval "
\$fids = \Drupal::entityQuery('file')->condition('status', 0)->accessCheck(FALSE)->execute();
echo count(\$fids) . ' fichiers temporaires' . PHP_EOL;
"
```
