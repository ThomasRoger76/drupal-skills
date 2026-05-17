# Changelog — drupal-media

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
| v1.0 | D8.6+, D9, D10, D11 | Media core D8.6, Media Library stable D9, file_url_generator D9, WebP D10 |
