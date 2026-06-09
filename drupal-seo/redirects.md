---
name: drupal-seo — redirects
description: Gérer les redirections HTTP avec drupal/redirect - créer des 301/302, importer en masse, intégration Pathauto, et diagnostiquer les boucles.
---

# Redirections — drupal/redirect

## Installation

```bash
composer require drupal/redirect
drush en redirect -y
```

---

## Créer une Redirection Manuelle

```bash
# Via l'UI : /admin/config/search/redirect/add
# Source : /ancienne-url
# Destination : /nouvelle-url (ou URL externe)
# Status code : 301 (permanent) ou 302 (temporaire)

# Via Drush
drush php:eval "
\$redirect = \Drupal\redirect\Entity\Redirect::create([
  'redirect_source' => ['path' => 'ancienne-page', 'query' => []],
  'redirect_redirect' => ['uri' => 'internal:/nouvelle-page'],
  'status_code' => 301,
  'language' => 'und',
]);
\$redirect->save();
echo 'Redirection créée : ' . \$redirect->id();
"

# Vers une URL externe
drush php:eval "
\$redirect = \Drupal\redirect\Entity\Redirect::create([
  'redirect_source' => ['path' => 'old-blog', 'query' => []],
  'redirect_redirect' => ['uri' => 'https://nouveau-site.com/blog'],
  'status_code' => 301,
  'language' => 'und',
]);
\$redirect->save();
"
```

---

## Import en Masse depuis CSV

```bash
# Module redirect_import
composer require drupal/redirect_import
drush en redirect_import -y

# Format CSV (sans en-tête) :
# /ancienne-url,/nouvelle-url,301
# /old-page,/new-page,301
# /old-category/old-article,/new-category/new-article,301
# https://old-site.com/page,/page,301

# Import via UI : /admin/config/search/redirect/import
```

```php
// Import programmatique depuis un tableau
function import_redirects_from_array(array $redirects): int {
  $count = 0;
  foreach ($redirects as [$source, $destination, $code]) {
    // Éviter les doublons
    $existing = \Drupal::service('redirect.repository')
      ->findMatchingRedirect($source);
    if ($existing) {
      continue;
    }

    \Drupal\redirect\Entity\Redirect::create([
      'redirect_source' => ['path' => ltrim($source, '/'), 'query' => []],
      'redirect_redirect' => ['uri' => str_starts_with($destination, 'http') ? $destination : 'internal:' . $destination],
      'status_code' => (int) $code,
      'language' => 'und',
    ])->save();

    $count++;
  }
  return $count;
}
```

---

## Intégration avec Pathauto

```
/admin/config/search/path/settings → section "Update action" :
→ ✅ "Create a new alias. Delete the old alias and create a redirect."
   (libellé exact Pathauto — nécessite le module drupal/redirect activé)

Comportement :
  Article "Mon Titre" → alias /articles/mon-titre
  Renommé en "Mon Nouveau Titre" → alias /articles/mon-nouveau-titre
  Redirect auto : /articles/mon-titre → 301 → /articles/mon-nouveau-titre
```

---

## Diagnostiquer et Corriger les Boucles

```bash
# Tester une URL avec suivi de redirections
curl -L --max-redirs 5 -I https://mon-site.com/ancienne-url 2>&1 | grep -E "HTTP|Location"

# Boucle = HTTP/2 301 → Location: /ancienne-url → HTTP/2 301 → ...

# Trouver les boucles dans la DB
drush sql-query "
SELECT rs.path, rr.uri, r.status_code
FROM redirect r
JOIN redirect__redirect_source rs ON r.id = rs.entity_id
JOIN redirect__redirect_redirect rr ON r.id = rr.entity_id
WHERE rr.uri LIKE '%ancienne-url%'
OR rs.path LIKE '%ancienne-url%'
"

# Supprimer une redirection problématique
drush php:eval "
\$repo = \Drupal::service('redirect.repository');
\$redirect = \$repo->findMatchingRedirect('ancienne-url-en-boucle');
if (\$redirect) {
  \$redirect->delete();
  echo 'Redirection supprimée.';
}
"
```

---

## Purger les Redirections Obsolètes

```bash
# Supprimer les redirections qui pointent vers une URL qui n'existe plus
drush php:eval "
\$redirects = \Drupal::entityTypeManager()->getStorage('redirect')->loadMultiple();
foreach (\$redirects as \$redirect) {
  \$uri = \$redirect->getRedirectUrl()->toString();
  \$response = \Drupal::httpClient()->head(\$uri, ['http_errors' => false]);
  if (\$response->getStatusCode() === 404) {
    echo 'Orphelin : ' . \$redirect->getSourceUrl() . ' → ' . \$uri . PHP_EOL;
    // \$redirect->delete();  // Décommenter pour supprimer
  }
}
"

# Compter les redirections
drush php:eval "
echo \Drupal::entityTypeManager()->getStorage('redirect')->getQuery()->count()->execute() . ' redirections';
"
```

---

## Checklist Migration de Site

```
Avant la mise en ligne d'un nouveau site :

[ ] Exporter toutes les URLs de l'ancien site (sitemap.xml ou crawl)
[ ] Créer les redirections 301 pour chaque URL qui change
[ ] Tester 20 URLs représentatives avec curl -L
[ ] Vérifier qu'aucune boucle n'existe (curl --max-redirs 3)
[ ] Vérifier les redirections dans Google Search Console après indexation
[ ] Activer Pathauto "Redirect on update" pour les futures modifications
```
