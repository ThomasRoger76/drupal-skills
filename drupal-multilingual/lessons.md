# Leçons — drupal-multilingual

Bugs multilingues découverts en projet réel. Mis à jour après chaque incident.

---

## 2026-05-16 — Création du skill

### Champs affichant la langue d'origine malgré navigation en FR
- **Symptôme :** Sur un site FR/EN, les champs s'affichent en anglais même quand on navigue en français
- **Cause :** `$variables['node']` dans `hook_preprocess_node` est dans la langue d'origine — pas du contexte courant
- **Correct :** `$node = \Drupal::service('entity.repository')->getTranslationFromContext($node);` en début de preprocess
- **Prévention :** Sur tout site multilingue, TOUJOURS appeler `getTranslationFromContext()` dans les preprocess

### Cache context `languages:language_interface` manquant
- **Symptôme :** Drupal sert la même version cachée FR à des visiteurs EN (et vice-versa)
- **Cause :** Render array sans `'contexts' => ['languages:language_interface']` — une seule version mise en cache
- **Correct :** Ajouter `'#cache' => ['contexts' => ['languages:language_interface']]` sur tous les composants multilingues
- **Prévention :** Règle sur les sites multilingues : tout render array avec du texte traduit doit avoir ce context

### Views ne filtrant pas par langue
- **Symptôme :** La View affiche des articles EN sur la version FR du site
- **Cause :** La Views n'avait pas "Translation Language" configuré sur "Current user's language"
- **Correct :** Views → Advanced → Language → Content language et Table Language → "Interface text language selected for page"
- **Prévention :** Sur tout site multilingue, vérifier les settings de langue dans chaque View au moment de la création

### Config Translation désactivé → menus et blocs non traduits
- **Symptôme :** Les menus et blocs s'affichent en langue d'origine quelle que soit la langue de navigation
- **Cause :** Le module `config_translation` n'était pas activé — seul `content_translation` était actif
- **Correct :** `drush en config_translation -y` + traduire les menus/blocs dans `/admin/config/regional/config-translation`
- **Prévention :** Les 4 modules multilinguisme doivent être activés ensemble — vérifier avec `drush pm:list --status=enabled | grep -E "language|locale|config_translation|content_translation"`

### `drush locale:import` sans `--override=all` → traductions non mises à jour
- **Symptôme :** Mises à jour de traductions .po ne prennent pas effet
- **Cause :** Sans `--override=all`, seules les chaînes non encore traduites sont importées
- **Correct :** `drush locale:import fr fichier.po --override=all`
- **Prévention :** Pour les mises à jour de traductions : toujours `--override=all`

### Pathauto — alias URL identiques pour différentes langues
- **Symptôme :** `/article/mon-titre` pointe vers l'article EN et FR simultanément
- **Cause :** Pathauto non configuré par langue — génère le même alias pour toutes les traductions
- **Correct :** Configurer des patterns Pathauto séparés par langue avec `[node:language]` dans le pattern ou dans les conditions
- **Prévention :** Sur site multilingue avec Pathauto : vérifier que chaque langue a son propre pattern ou préfixe

### EntityQuery `->condition('langcode', 'fr')` — résultats manquants
- **Symptôme :** Une requête cherchant des nœuds FR ne retourne rien même si des traductions FR existent
- **Cause :** Certains nœuds ont `langcode = 'en'` (langue d'origine) mais ont une traduction FR — ils ne sont pas retournés par ce filtre
- **Correct :** Pour chercher les nœuds QUI ONT une traduction FR (quelle que soit leur langue d'origine) : utiliser une join spécifique ou passer par Views avec Translation Language. Pour ne cibler QUE les originaux : ajouter `->condition('default_langcode', 1)`.
- **Prévention :** `->condition('langcode', 'fr')` filtre sur le langcode de l'original, pas des traductions. Pour les traductions, utiliser Views ou une query SQL directe sur la table de traduction. Voir le bloc dédié dans content-translation.md.

---

## 2026-06-09 — Audit qualité (v1.1)

### Config de négociation de langue inexacte (`language.negotiation.yml`)
- **Symptôme :** Un `drush cim` échoue ou n'applique pas la négociation — le fichier `language.negotiation.yml` est ignoré
- **Cause :** Ce fichier n'existe pas dans Drupal core. La négociation s'écrit dans `language.types.yml` (clés `configurable` + `negotiation.<type>.enabled` / `method_weights`, méthodes nommées `language-url`, `language-session`...) et `language.negotiation.url.yml` (`source: path_prefix|domain`, `prefixes`, `domains`).
- **Correct :** Voir multilingual-setup.md et language-negotiation.md (structure YAML corrigée).
- **Prévention :** Toujours vérifier le nom réel d'un fichier de config avec `docker compose exec php drush config:get <name>` avant de l'écrire à la main.

### Commandes drush sans préfixe Docker
- **Symptôme :** `drush: command not found` ou exécution sur le mauvais environnement (host au lieu du conteneur)
- **Cause :** Drush n'est disponible que dans le conteneur PHP — les commandes doivent passer par `docker compose exec php drush ...`
- **Prévention :** Standard projet — JAMAIS `ddev`, JAMAIS `drush` nu. Toujours `docker compose exec php drush`.

### TMGMT — assignation directe `$job->translator = ...`
- **Symptôme :** Le traducteur n'est pas pris en compte sur le job
- **Cause :** `tmgmt_job` est une entité ; les champs se définissent via `set()`, pas par accès direct à une propriété magique
- **Correct :** `$job->set('translator', 'deepl');` et `$job->set('settings', []);`
