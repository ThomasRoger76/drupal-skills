# Changelog — drupal-media

---

## v1.1 — 2026-06-09

**Audit qualité D11 — mise à niveau des API dépréciées et des attributs de plugins**

### Corrigé

- **`media-programmatic.md`** — `system_retrieve_file()` (déprécié D10.3+) remplacé
  par `httpClient()->get()` + `file.repository->writeData()`. Suppression du pattern
  `File::create()` + `setFileUri()` qui ne déplace pas le fichier physique.
  Détection des médias orphelins corrigée : `file.usage` (faux négatifs systématiques)
  → `drupal/entity_usage` `listSources()`.
- **`media-library.md`** — `FileSystemInterface::EXISTS_REPLACE` (déprécié D10.3)
  → enum `FileExists::Replace` via `file.repository->copy()`. `image_path_flush()`
  (non public) et `drush image-flush` → `drush image:flush` / `ImageStyle::flush()`.
- **`media-types.md`** — Annotation `@MediaSource` → attribut PHP `#[MediaSource]`
  (D10.2+, annotation supprimée D12) avec `new TranslatableMarkup()`.
- **`media-migration.md`** — Annotation `@MigrateSource` → attribut PHP
  `#[MigrateSource]` ; import `Row` inutilisé retiré.
- **`SKILL.md`** — 5 anti-patterns ajoutés (system_retrieve_file, FileExists,
  attributs PHP, File::create incomplet, file.usage orphelins). Table d'évolution
  enrichie (FileExists, attributs PHP, system_retrieve_file). Orphelins :
  `entity_usage` au lieu de `media_entity_file_replace` (rôle erroné).
- **`lessons.md`** — 4 incidents D11 documentés.

---

## v1.0 — 2026-05-16

**Création initiale — skill manquant identifié lors de l'audit ultra-critique**

### Couverture

**`SKILL.md`**
- Tableau comparatif File vs Image vs Media (9 critères)
- Quick Decision Table (20+ entrées)
- Anti-patterns critiques (7 entrées)
- Table versioning D8→D11 (Media Library core D9, WebP D10, file_url_generator)

**`media-types.md`**
- Media types natifs Drupal (Image, Remote video, Document, Video, Audio)
- Configuration YAML complète d'un Media type
- Remote video oEmbed (YouTube, Vimeo) + whitelist providers
- Champs custom sur un Media type
- Formatters de Media (image style, responsive image, URL)
- Media Source Plugin custom (API externe)

**`media-programmatic.md`**
- `Media::load()`, `loadMultiple()`, EntityQuery
- Créer un Media depuis un fichier (File entity + Media entity)
- `file_url_generator` service — URL absolue, relative, avec image style
- Accès aux métadonnées d'image (alt, title, width, height)
- Preprocess — exposer les données Media au template avec cache correct
- JSON:API avec includes Media (`?include=field_image.field_media_image`)
- Nettoyage des médias orphelins via `file.usage`

**`lessons.md`**
- 6 incidents Media réels avec corrections

---

## Compatibilité Drupal

| Skill version | Drupal | Notes |
|--------------|--------|-------|
| v1.1 | D10.2+, D11 | Attributs PHP plugins (D10.2), enum FileExists (D10.3), system_retrieve_file déprécié |
| v1.0 | D8.6+, D9, D10, D11 | Media core D8.6, Media Library stable D9, file_url_generator D9, WebP D10 |
