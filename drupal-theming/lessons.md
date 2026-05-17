# Leçons — drupal-theming

Bugs et patterns découverts en usage réel. Mis à jour après chaque correction.

---

## Comment ajouter une leçon

Après chaque bug issu d'un projet utilisant ce skill :
1. Identifier si la cause racine est un gap dans `drupal-theming`
2. Ajouter une entrée ci-dessous avec date + symptôme + cause + prévention
3. Corriger le fichier source concerné
4. Ajouter une ligne dans `CHANGELOG.md`

Format : `date | composant | symptôme | cause | prévention`

---

## 2026-05-14 — Création du skill

Pas encore de bugs en usage réel. Les points suivants sont des pièges connus documentés dès la v1 :

### `class="{{ attributes.class }} ma-classe"` — écrase les classes Drupal
- **Symptôme :** Les classes système Drupal (block--id, node--type-article, etc.) disparaissent
- **Cause :** L'écriture directe de `attributes.class` retourne un objet, pas une string — et écrase le contexte
- **Correct :** `{{ attributes.addClass('ma-classe') }}`
- **Prévention :** Règle absolue : ne JAMAIS interpoler `attributes.class` comme string

### `core/jquery` en dépendance — jQuery absent en D9+
- **Symptôme :** `$ is not defined` / `jQuery is not defined` sur D9/D10/D11
- **Cause :** jQuery n'est plus chargé par défaut depuis D9
- **Correct :** Utiliser `core/once` + vanilla JS + pattern `Drupal.behaviors`
- **Prévention :** Ne jamais ajouter `core/jquery` sans besoin explicite — vérifier si la logique peut être en vanilla JS

### `base theme: classy` sur D10
- **Symptôme :** Erreur fatale à l'activation du thème sur D10+
- **Cause :** Module `classy` supprimé en D10
- **Correct :** `base theme: stable9` ou `base theme: false`
- **Prévention :** Vérifier la table versioning dans SKILL.md avant tout upgrade de thème

### Template non trouvé malgré bon nom de fichier
- **Symptôme :** Le template Twig n'est pas pris en compte
- **Cause :** Les underscores `_` et tirets `-` sont interchangeables dans les noms de fichiers Twig — mais le debug indique le nom exact avec les underscores DANS les commentaires HTML. Les fichiers TWIG utilisent des tirets `-`.
  - Bonne pratique : `node--article--full.html.twig` (tirets)
  - Commentaire debug : `node__article__full.html.twig` (underscores) — c'est le nom du hook, pas du fichier
- **Prévention :** Toujours copier le nom depuis le commentaire debug HTML et REMPLACER `__` par `--`

### `file_create_url()` supprimé D10 — Fatal Error
- **Composant :** responsive-images.md
- **Symptôme :** `Call to undefined function file_create_url()` sur D10+
- **Cause :** Fonction supprimée en D10 après déprecation en D9
- **Correct :** `\Drupal::service('file_url_generator')->generateAbsoluteString($uri)`
- **Prévention :** Chercher `file_create_url` avant tout upgrade D9→D10

### `{{ is_front_page }}` inexistante dans Twig
- **Composant :** twig-templates.md
- **Symptôme :** Variable non définie dans les templates Twig
- **Cause :** La variable s'appelle `is_front` (injectée par `template_preprocess_page()`)
- **Correct :** `{% if is_front %}`
- **Prévention :** `dump()` pour vérifier les noms exacts des variables disponibles

### `views-fields--VIEW-ID.html.twig` n'existe pas
- **Composant :** theme-suggestions.md
- **Symptôme :** Template non pris en compte, Twig debug ne le montre pas
- **Cause :** Tous les templates Views commencent par `views-view-*`, pas `views-*`
- **Correct :** `views-view-fields--VIEW-ID--DISPLAY-ID.html.twig`
- **Prévention :** Toujours copier le nom exact depuis le commentaire Twig debug HTML

### `libraries-override` en doublon dans `.info.yml` — YAML invalide
- **Composant :** theme-anatomy.md / libraries-assets.md
- **Symptôme :** Seule la dernière occurrence de `libraries-override:` est prise en compte
- **Cause :** YAML n'accepte pas deux clés identiques au même niveau — la seconde écrase la première
- **Correct :** Regrouper toutes les overrides sous UNE SEULE clé `libraries-override:`
- **Prévention :** Écrire toutes les overrides dans le même bloc, pas dans des sections séparées

### `base theme: bootstrap5` — module contrib requis, jQuery activé
- **Symptôme :** Thème ne démarre pas avec "Base theme bootstrap5 not found"
- **Cause :** `bootstrap5` est un base theme contrib, pas un core theme — nécessite `composer require drupal/bootstrap5`
- **Correct :** `composer require drupal/bootstrap5 && docker compose exec php drush en bootstrap5 -y`
- **Prévention :** Si `base theme: bootstrap5`, jQuery est activé par défaut — ajouter `core/jquery` dans les librairies est acceptable mais non obligatoire

### `core/drupal.once` — Alias invalide dans `.libraries.yml`
- **Symptôme :** `once()` non disponible en JS malgré la déclaration dans libraries.yml
- **Cause :** `core/drupal.once` n'est pas le bon identifiant de librairie
- **Correct :** Utiliser `core/once` (sans le préfixe `drupal.`)
- **Prévention :** Les librairies core valides : `core/drupal`, `core/once`, `core/jquery`, `core/drupalSettings`

### Preprocess multilingue — `$variables['node']` non traduit
- **Symptôme :** Les champs affichent la valeur en langue d'origine même en naviguant en FR
- **Cause :** `$variables['node']` dans le preprocess est le nœud dans sa langue d'origine
- **Correct :** `$node = \Drupal::service('entity.repository')->getTranslationFromContext($node);`
- **Prévention :** Sur tout site multilingue, toujours appliquer `getTranslationFromContext()` + ajouter `'languages:language_interface'` au cache contexts

### `drush cr` insuffisant après modification du `.theme`
- **Symptôme :** Les changements de preprocess ne prennent pas effet
- **Cause :** Le cache Twig ET le cache des fonctions PHP sont indépendants — parfois `drush cr` ne suffit pas
- **Correct :** `docker compose exec php drush cr` puis si nécessaire redémarrer le container PHP ou activer Twig debug
- **⚠️ Ne jamais utiliser en production :** `deleteAll()` sur le cache render efface le cache de tous les utilisateurs simultanément — réservé au dev uniquement
- **Prévention :** En dev, activer `auto_reload: true` et `cache: false` dans `services.yml` — le cache Twig se recharge automatiquement
