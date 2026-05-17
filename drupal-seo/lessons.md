# Leçons — drupal-seo

Problèmes SEO découverts en projets Drupal réels. Mis à jour après chaque incident.

---

## Comment ajouter une leçon

Après chaque problème SEO découvert :
1. Identifier si le skill aurait pu prévenir l'erreur
2. Ajouter une entrée avec symptôme, cause, correction, prévention
3. Ajouter une ligne dans CHANGELOG.md

---

### 2026-05-16 — `noindex` laissé actif en production — site désindexé

- **Symptôme :** Trafic organique s'effondre après un déploiement. Google Search Console signale "bloqué par robots meta tag".
- **Cause :** `<meta name="robots" content="noindex">` activé sur l'environnement de développement, copié vers la production lors du `drush cim`
- **Correct :** Supprimer immédiatement le tag noindex en production. Attendre la prochaine crawl Google (1-7 jours).
- **Prévention :** Dans `settings.php` production : `$config['metatag.metatag_defaults.global']['tags']['robots'] = 'index, follow'`. Ajouter une vérification dans la checklist de déploiement.

### 2026-05-16 — Meta descriptions identiques — pénalité duplicate content

- **Symptôme :** Google Search Console signale "Duplicate meta descriptions". Le trafic organique est plat sur toutes les pages.
- **Cause :** La configuration Metatag globale utilise un texte fixe au lieu du token `[node:summary]`
- **Correct :** Configurer par type de contenu : `[node:summary]` (résumé saisi) ou `[node:body:summary]` (300 premiers chars du body)
- **Prévention :** Vérifier les descriptions uniques dans Google Search Console sous "HTML improvements"

### 2026-05-16 — Alias Pathauto non régénérés après migration — node/42 partout

- **Symptôme :** Après une migration de contenu, toutes les URLs sont `node/42`, `node/43`...
- **Cause :** `drush migrate:import` n'appelle pas automatiquement Pathauto. Les aliases ne sont pas générés.
- **Correct :** `drush pathauto:aliases-generate` après chaque import de contenu
- **Prévention :** Ajouter dans le script de migration : `drush pathauto:aliases-generate node article`

### 2026-05-16 — Redirections perdues après migration de base de données — boucles infinies

- **Symptôme :** Certaines URLs retournent une redirection infinie (301 → 301 → ...) après migration
- **Cause :** La table `redirect` a été migrée depuis un environnement de staging avec des URLs hardcodées pointant vers staging
- **Correct :** `drush sql-query "DELETE FROM redirect WHERE redirect_redirect__uri LIKE '%staging%'"` puis récréer les redirections correctes
- **Prévention :** Vérifier les redirections post-migration avec `curl -L --max-redirs 5 URL`

### 2026-05-16 — Open Graph image absente — partage social cassé

- **Symptôme :** Partage LinkedIn/Facebook d'un article ne montre pas l'image de l'article
- **Cause :** Le token Metatag `[node:field_image:entity:file:url]` retourne vide si le champ Image reference un Media plutôt qu'un File direct
- **Correct :** Utiliser le bon token pour Media : `[node:field_image:entity:field_media_image:entity:url]`
- **Prévention :** Tester le partage social avec https://developers.facebook.com/tools/debug/ pour chaque type de contenu après configuration initiale

### 2026-05-16 — Sitemap XML non soumis à Google — pages non indexées

- **Symptôme :** Nouvelles pages non indexées après 2 semaines de mise en ligne
- **Cause :** Sitemap généré mais jamais soumis à Google Search Console
- **Correct :** `/admin/config/search/simplesitemap` → copier l'URL du sitemap → Google Search Console → Sitemaps → Submit
- **Prévention :** Checklist de mise en production inclure "Soumettre sitemap à GSC"

### 2026-05-16 — Schema.org JSON-LD invalide — rich snippets non activés

- **Symptôme :** Google Rich Results Test montre des erreurs. Pas de rich snippets dans les SERPs.
- **Cause :** JSON généré avec des valeurs nulles ou des types incorrects (timestamp Unix au lieu de date ISO 8601)
- **Correct :** `date('c', $node->getCreatedTime())` pour les dates (ISO 8601). Valider sur schema.org/validator
- **Prévention :** Toujours valider le JSON-LD sur https://validator.schema.org/ après implémentation

### 2026-05-16 — Pathauto translitération désactivée — accents dans les URLs

- **Symptôme :** URL générée : `/articles/léçon-de-drupal` (avec caractères spéciaux) → 404 sur certains serveurs
- **Cause :** L'option "Transliterate prior to creating alias" était désactivée dans Pathauto settings
- **Correct :** `/admin/config/search/path/settings` → activer "Transliterate" → régénérer tous les aliases
- **Prévention :** Activer la translitération dès la configuration initiale de Pathauto
