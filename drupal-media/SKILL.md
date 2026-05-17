---
name: drupal-media
description: Use when managing Drupal Media Library, configuring Media types and bundles (image, video, document, remote_video, audio), using Media entity reference fields, implementing focal point with drupal/focal_point, configuring image styles and responsive images for media, handling private media files with access control, translating media entities, managing media in migrations (migrate source plugins for files and media), creating custom Media source plugins, bulk uploading media, configuring Media oEmbed for remote videos (YouTube, Vimeo), accessing media programmatically (Media::create, media_entity), or setting up media library widget configuration in Drupal 8-11+
---

# Drupal Media — Référence Complète

## Overview

Référentiel complet du système Media Drupal 8-11+ (core depuis D8.6) : Media Library, Media Types, sources oEmbed, accès programmatique, traduction, migrations, et gestion des fichiers privés. Media remplace les champs File/Image directs pour les médias réutilisables.

## 🎯 La Règle Fondamentale

> **Media avant File direct.** Un champ Media entity reference est toujours préférable à un champ File ou Image direct — la media est une entité réutilisable, traductible, avec ses propres champs et permissions. Un champ File/Image ne peut être utilisé qu'à un seul endroit.

---

## Quick Decision Table

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Galerie d'images réutilisables entre nœuds | Media type "Image" + Entity Reference | [media-types.md](media-types.md) |
| Vidéo YouTube/Vimeo intégrée | Media type "Remote video" (oEmbed) | [media-types.md](media-types.md) |
| Document PDF téléchargeable | Media type "Document" | [media-types.md](media-types.md) |
| Fichier audio (podcast, musique) | Media type "Audio" | [media-types.md](media-types.md) |
| Vidéo hébergée localement | Media type "Video" + champ File | [media-types.md](media-types.md) |
| Bibliothèque de médias pour les éditeurs | Media Library (core D8.6+) | [media-library.md](media-library.md) |
| Upload de médias en masse | Media Library bulk upload | [media-library.md](media-library.md) |
| Recadrage automatique de l'image | `drupal/focal_point` | [media-library.md](media-library.md) |
| Accéder à un Media depuis PHP | `Media::load()`, `MediaInterface` | [media-programmatic.md](media-programmatic.md) |
| Créer un Media depuis PHP | `Media::create([...])` | [media-programmatic.md](media-programmatic.md) |
| Obtenir l'URL d'une image Media | `file_url_generator->generateAbsoluteString()` | [media-programmatic.md](media-programmatic.md) |
| Média privé (accès restreint) | `private://` + `hook_file_download()` | [media-access.md](media-access.md) |
| Accès conditionnel à un fichier | `hook_file_download()` avec vérification | [media-access.md](media-access.md) |
| Traduire une media (alt, title, champs) | `content_translation` sur le bundle media | [media-programmatic.md](media-programmatic.md) |
| Migrer des images depuis D7 | `migrate_plus` + source `d7_file` | [media-migration.md](media-migration.md) |
| Migrer des fichiers depuis CSV/dossier local | Plugin source `DirectoryIterator` | [media-migration.md](media-migration.md) |
| Source Media custom (API externe, S3...) | `MediaSourceBase` plugin custom | [media-types.md](media-types.md) |
| Image style (resize, crop, WebP) | `/admin/config/media/image-styles` | [media-library.md](media-library.md) |
| Responsive images (srcset) | `breakpoints.yml` + Responsive Image Style | [media-library.md](media-library.md) |
| MediaWidget dans un formulaire custom | `entity_reference` field avec Media Library widget | [media-programmatic.md](media-programmatic.md) |
| Référencer un Media depuis JSON:API | `?include=field_image.field_media_image` | [media-programmatic.md](media-programmatic.md) |
| Nettoyer les médias orphelins | Module `media_entity_file_replace` ou cron custom | [media-programmatic.md](media-programmatic.md) |
| **Recadrage manuel par le rédacteur** | `drupal/image_widget_crop` — crop zones configurables | [media-library.md](media-library.md) |
| **Images SVG nativement dans Drupal** | `drupal/svg_image` — render les SVG inline ou via `<img>` | [media-types.md](media-types.md) |
| Téléchargement de média avec bouton | `drupal/media_entity_download` — ajoute un lien download | [media-library.md](media-library.md) |

## Différences File vs Image vs Media

| Critère | File field | Image field | Media entity reference |
|---------|-----------|-------------|----------------------|
| Réutilisable entre entités | ❌ | ❌ | ✅ |
| Champs custom (légende, auteur...) | ❌ | ❌ | ✅ |
| Traduction | ❌ | ❌ | ✅ |
| Media Library | ❌ | ❌ | ✅ |
| Bulk upload | ❌ | ❌ | ✅ |
| oEmbed (YouTube...) | ❌ | ❌ | ✅ |
| Focal Point | ❌ | ⚠️ | ✅ |
| Révisions | ❌ | ❌ | ✅ |
| JSON:API normalisé | ❌ | ❌ | ✅ |
| **Recommandé** | Legacy | Legacy | **✅ Standard D9+** |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| Champ File/Image direct pour des médias réutilisables | Entity Reference vers Media | Pas de réutilisation, pas de Media Library |
| `file_create_url()` en D10+ | `file_url_generator->generateAbsoluteString()` | Fonction supprimée D10 |
| Stocker des fichiers confidentiels dans `public://` | `private://` + `hook_file_download()` | Accès public à des fichiers privés |
| `Media::load()` en boucle sur des listes | `loadMultiple()` | N+1 queries |
| Chemin de fichier hardcodé | `$file->getFileUri()` | Dépend de l'environnement |
| oEmbed sans whitelist de domaines | Configurer `oEmbed providers` | SSRF possible |
| Media non traduit sur site multilingue | Activer content_translation sur Media | Alt text en mauvaise langue |

## Évolution par Version Majeure

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| Media module (core) | ✅ D8.6+ | ✅ | ✅ | ✅ |
| Media Library (core) | ❌ contrib | ✅ stable | ✅ | ✅ |
| oEmbed (YouTube, Vimeo) | ✅ | ✅ | ✅ | ✅ |
| WebP dans Image Styles | ❌ | ❌ | ✅ | ✅ |
| `file_create_url()` | ✅ | ⚠️ deprecated | ❌ supprimé | ❌ |
| `file_url_generator` service | ❌ | ✅ | ✅ standard | ✅ |
| Focal Point (contrib) | ✅ | ✅ | ✅ | ✅ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Bugs Media découverts en projet.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions.

## See Also

- `drupal-core` — Entity API, champs Entity Reference, `file_url_generator`
- `drupal-security` — Fichiers privés, `hook_file_download()`, accès restreint
- `drupal-content-modeling` — Champ Image vs File vs Media — décision architecture
- `drupal-theming` — Templates Media, responsive images, WebP
- `drupal-migration` — Migrer des fichiers et médias depuis D7/CSV
- `drupal-api` — JSON:API avec Media entities (`?include=field_image.field_media_image`)
- `drupal-multilingual` — Traduction des Media entities
