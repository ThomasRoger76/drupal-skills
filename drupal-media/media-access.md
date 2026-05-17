---
name: drupal-media — access control
description: Contrôle d'accès aux fichiers Media Drupal - fichiers privés (private://), hook_file_download(), permissions par rôle, et flux de téléchargement sécurisé.
---

# Media Access Control — Fichiers Privés

## Public vs Privé

```
public://   → /sites/default/files/
            → URL directe accessible sans authentification
            → Idéal : images publiques, assets CSS/JS
            ❌ NE PAS utiliser pour : documents confidentiels, fichiers réservés aux membres

private://  → chemin hors webroot (ex: /var/private/)
            → Routé via Drupal → hook_file_download() vérifie les droits
            → Idéal : PDF réservés aux membres, documents légaux, exports
```

---

## Configurer le Répertoire Privé

```php
// settings.php — définir le chemin privé (HORS du webroot)
$settings['file_private_path'] = '/var/www/private';
// OU relatif au projet
$settings['file_private_path'] = DRUPAL_ROOT . '/../private';
```

```bash
# Créer le répertoire et les permissions
mkdir -p /var/www/private
chown www-data:www-data /var/www/private
chmod 750 /var/www/private

# Vérifier depuis Drupal
drush php:eval "echo \Drupal::config('system.file')->get('default_scheme');"
drush php:eval "echo \Drupal\Core\StreamWrapper\PrivateStream::basePath();"
```

---

## Configurer un Media Type pour Utiliser private://

```yaml
# config/install/field.storage.media.field_media_document.yml
langcode: fr
status: true
id: media.field_media_document
field_name: field_media_document
entity_type: media
type: file
settings:
  uri_scheme: private   # ← private:// au lieu de public://
  file_directory: 'documents/[date:custom:Y]-[date:custom:m]'
  file_extensions: 'pdf doc docx xls xlsx'
  max_filesize: '20 MB'
```

```
Interface : /admin/structure/media/manage/document/fields/media.document.field_media_document
→ Field settings → Upload destination → Private files
```

---

## hook_file_download() — Contrôle d'Accès Fin

```php
<?php

/**
 * Implements hook_file_download().
 *
 * Contrôler l'accès aux fichiers privés.
 * Retourner : headers array (accès autorisé) | -1 (accès refusé) | NULL (inconnu)
 */
function mon_module_file_download(string $uri): array|int|null {
  // Uniquement pour les fichiers dans notre répertoire privé
  if (!str_starts_with($uri, 'private://documents/')) {
    return NULL;  // On ne gère pas ce fichier
  }

  $current_user = \Drupal::currentUser();

  // Accès public → jamais (private = toujours authentifié)
  if ($current_user->isAnonymous()) {
    return -1;  // 403 Forbidden
  }

  // Vérifier une permission Drupal
  if (!$current_user->hasPermission('download private documents')) {
    return -1;
  }

  // Logique métier : vérifier si l'utilisateur a accès à CE fichier spécifique
  $file = \Drupal::service('file.repository')->loadByUri($uri);
  if (!$file) {
    return -1;
  }

  // Vérifier via l'entité Media associée
  $media_storage = \Drupal::entityTypeManager()->getStorage('media');
  $mids = $media_storage->getQuery()
    ->condition('field_media_document.target_id', $file->id())
    ->accessCheck(FALSE)
    ->execute();

  if (!$mids) {
    return NULL;  // Pas de Media associé — laisser Drupal décider
  }

  $media = $media_storage->load(reset($mids));
  if (!$media || !$media->access('view', $current_user)) {
    return -1;
  }

  // Accès autorisé — retourner les headers HTTP
  return [
    'Content-Type' => $file->getMimeType(),
    'Content-Length' => $file->getSize(),
    // Forcer le téléchargement (au lieu de l'affichage dans le navigateur)
    'Content-Disposition' => 'attachment; filename="' . $file->getFilename() . '"',
    // Pas de cache pour les fichiers privés
    'Cache-Control' => 'private',
    'Pragma' => 'no-cache',
  ];
}
```

---

## Accès par Nœud — Exemple Multi-Tenant

```php
/**
 * Implements hook_file_download().
 *
 * Les fichiers attachés à un nœud ne sont téléchargeables
 * que si l'utilisateur a accès à ce nœud.
 */
function mon_module_file_download(string $uri): array|int|null {
  if (!str_starts_with($uri, 'private://')) {
    return NULL;
  }

  $current_user = \Drupal::currentUser();
  $file = \Drupal::service('file.repository')->loadByUri($uri);

  if (!$file) {
    return NULL;
  }

  // Trouver tous les nœuds qui utilisent ce fichier
  $usage = \Drupal::service('file.usage')->listUsage($file);

  if (isset($usage['file']['node'])) {
    foreach (array_keys($usage['file']['node']) as $nid) {
      $node = \Drupal::entityTypeManager()->getStorage('node')->load($nid);
      if ($node && $node->access('view', $current_user)) {
        return [
          'Content-Type' => $file->getMimeType(),
          'Content-Length' => $file->getSize(),
        ];
      }
    }
    return -1;  // Nœud existe mais pas d'accès
  }

  return NULL;  // Fichier non associé à un nœud
}
```

---

## Permissions Media par Rôle

```
/admin/people/permissions → section "Media"

Permissions disponibles :
  ├── "Administer media" → gérer tous les types de media
  ├── "View media" → voir les médias publiés (anonymes par défaut)
  ├── "View own unpublished media" → voir ses brouillons
  ├── "Create IMAGE media" → uploader des images
  ├── "Edit own IMAGE media" → modifier ses propres médias
  ├── "Edit any IMAGE media" → modifier tous les médias
  ├── "Delete own IMAGE media"
  └── "Delete any IMAGE media"
```

---

## Protéger le Répertoire public// (Fichiers PHP)

```apache
# web/sites/default/files/.htaccess (généré automatiquement par Drupal)
# Bloquer l'exécution PHP dans public://
php_flag engine off

# Vérifier que .htaccess est en place
ls -la web/sites/default/files/.htaccess

# Régénérer si supprimé
drush php:eval "file_ensure_htaccess();"
```

---

## Audit Sécurité des Médias

```bash
# Vérifier les fichiers PHP dans public://
find web/sites/default/files -name "*.php" -o -name "*.phtml" -o -name "*.phar"

# Vérifier que private:// est bien hors du webroot
drush php:eval "echo \Drupal\Core\StreamWrapper\PrivateStream::basePath();"
# Doit être HORS de web/ (ex: /var/www/private ou ../private)

# Fichiers dans public:// qui devraient être privés ?
find web/sites/default/files -name "*.pdf" | head -20
# Si des PDFs confidentiels → les migrer vers private://
```
