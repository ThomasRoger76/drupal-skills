# Leçons — drupal-media

Bugs Media rencontrés en projets Drupal réels.

---

## 2026-05-16 — Création du skill

### `file_create_url()` Fatal Error en D10
- **Symptôme :** `Call to undefined function file_create_url()` après upgrade D9→D10
- **Cause :** Fonction supprimée en D10, dépréciée en D9
- **Correct :** `\Drupal::service('file_url_generator')->generateAbsoluteString($uri)`
- **Prévention :** Chercher `file_create_url` avant tout upgrade D9→D10 — `grep -r "file_create_url" web/modules/custom/`

### `alt` obligatoire non configuré — Validation échoue
- **Symptôme :** Upload d'image impossible via le formulaire avec "Le champ Alt est obligatoire"
- **Cause :** `alt_field_required: true` dans la config du champ mais le formulaire ne l'affiche pas correctement
- **Correct :** Vérifier la config du display du Media type (`/admin/structure/media/manage/image/form-display`)
- **Prévention :** Toujours tester l'upload d'image dans tous les contextes (formulaire de nœud, Media Library standalone)

### Media orphelin après suppression de nœud
- **Symptôme :** Des centaines de médias inutilisés dans la Media Library
- **Cause :** La suppression d'un nœud ne supprime pas les medias référencés (comportement voulu par Drupal)
- **Correct :** Module `drupal/media_entity_file_replace` ou cron de nettoyage custom via `file.usage`
- **Prévention :** Décider en amont de la politique de nettoyage des médias orphelins

### N+1 queries sur les medias dans preprocess
- **Symptôme :** Vue d'une liste de 20 articles = 20+ requêtes pour les images
- **Cause :** `$node->get('field_image')->entity` dans le preprocess chargé pour chaque nœud séparément
- **Correct :** `loadMultiple()` de tous les Media IDs collectés avant le rendu
- **Prévention :** Jamais `->entity` en boucle sur un champ Media — batch loading obligatoire

### Media privé accessible sans authentification
- **Symptôme :** Des fichiers marqués "private" sont accessibles via leur URL directe
- **Cause :** `hook_file_download()` non implémenté — Drupal ne vérifie pas l'accès par défaut
- **Correct :** Implémenter `hook_file_download()` avec vérification des permissions
- **Prévention :** Tout Media dans `private://` doit avoir un `hook_file_download()` correspondant

### Alt text en langue d'origine sur site multilingue
- **Symptôme :** L'alt text des images est toujours en anglais même en naviguant en français
- **Cause :** `$media->get('field_media_image')->alt` retourne l'alt de la langue d'origine
- **Correct :** `$media = entity.repository->getTranslationFromContext($media)` avant d'accéder aux champs
- **Prévention :** Activer `content_translation` sur le Media type + `getTranslationFromContext()` dans les preprocess
