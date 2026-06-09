# Leçons — drupal-token

Problèmes avec les tokens Drupal découverts en projet réel.

---

### 2026-05-16 — Token `[node:field_image:entity:file:url]` retourne vide avec Media

- **Symptôme :** L'URL de l'image dans Metatag est vide bien que le nœud ait une image
- **Cause :** Le champ Image est de type "Entity Reference → Media" (pas "File" direct). Le chemin du token est différent.
- **Correct :** `[node:field_image:entity:field_media_image:entity:url]` (passe par le Media entity puis le File)
- **Prévention :** Tester les tokens dans "Browse available tokens" avant de les utiliser en production

### 2026-05-16 — Token non résolu visible dans le texte — `[node:field_optionnel]` affiché

- **Symptôme :** La meta description affiche littéralement `[node:field_resume]` pour les articles sans résumé
- **Cause :** `['clear' => FALSE]` (défaut) laisse les tokens non résolus dans le texte
- **Correct :** `token->replace($text, $data, ['clear' => TRUE])` — supprime les tokens vides
- **Prévention :** Toujours utiliser `['clear' => TRUE]` OU définir un fallback dans le token chain `[node:field_resume|node:body:summary]`

### 2026-05-16 — `hook_tokens()` pas appelé — type incorrect

- **Symptôme :** Le token custom `[mon_type:ma_valeur]` n'est jamais remplacé
- **Cause :** Le type déclaré dans `hook_token_info()` (`mon_module_commande`) ne correspond pas au type passé dans le contexte (`$data['commande']`)
- **Correct :** Le type dans `hook_token_info()` doit matcher la clé de `$data` dans `hook_tokens()`
- **Prévention :** Vérifier les types avec `drush php:eval "var_dump(array_keys(\Drupal::service('token')->getInfo()['tokens']));"` 

### 2026-05-16 — `token_replace()` fonction D7 — erreur PHP en D10+

- **Symptôme :** `Call to undefined function token_replace()` après upgrade
- **Cause :** Ancienne fonction D7 supprimée en D10
- **Correct :** `\Drupal::service('token')->replace($text, $data, ['clear' => TRUE])`
- **Prévention :** Chercher `token_replace(` avant tout upgrade D9→D10

### 2026-05-16 — Token Pathauto avec caractères spéciaux — URL illisible

- **Symptôme :** URL générée : `/articles/l%C3%A9%C3%A7on-drupal` au lieu de `/articles/lecon-drupal`
- **Cause :** Module Token installé mais option "Transliterate prior to creating alias" désactivée dans Pathauto
- **Correct :** `/admin/config/search/path/settings` → activer la translitération → régénérer les alias
- **Prévention :** Activer la translitération dès l'installation de Pathauto — avant la création du premier contenu

### 2026-05-16 — Token date avec format incorrect — Schema.org invalide

- **Symptôme :** Google Search Console signale des dates Schema.org invalides
- **Cause :** `[node:created]` retourne une date lisible ("15 mai 2026") au lieu du format ISO 8601 requis par Schema.org
- **Correct :** Utiliser `[node:created:html_datetime]` qui retourne `2026-05-15T10:30:00+02:00`
- **Prévention :** Pour Schema.org, toujours utiliser le format `html_datetime` sur les dates

### 2026-06-08 — Token custom : contenu périmé après modification de l'entité

- **Symptôme :** Une méta description générée par un token custom `[commande:montant]` reste figée après mise à jour de la commande ; il faut un `drush cr` pour la rafraîchir.
- **Cause :** `hook_tokens()` lisait l'entité sans propager ses cache tags. Le rendu (page/bloc) est mis en cache sans dépendance vers l'entité → jamais invalidé.
- **Correct :** Recevoir le 5e paramètre `BubbleableMetadata $bubbleable_metadata` et appeler `$bubbleable_metadata->addCacheableDependency($entity)` pour chaque entité lue. Passer aussi ce paramètre à `$token_service->generate(...)` lors d'un chaining.
- **Prévention :** Toujours typer la signature complète de `hook_tokens()` (D8.6+) ; tout token dérivé d'une entité = au moins un `addCacheableDependency()`.

### 2026-06-08 — module_load_include() déprécié pour charger le .tokens.inc

- **Symptôme :** Avertissement de dépréciation `module_load_include()` en D10.3+/D11.
- **Cause :** Fonction procédurale dépréciée. De plus, `mon_module.tokens.inc` est auto-chargé par le système token — aucun include manuel n'est nécessaire.
- **Correct :** Placer `hook_token_info()`/`hook_tokens()` directement dans `mon_module.tokens.inc` (auto-chargé) ou `.module`. Si un include manuel est requis ailleurs : `\Drupal::moduleHandler()->loadInclude('mon_module', 'inc', 'mon_module.tokens')`.
- **Prévention :** Suivre la convention core (`node.tokens.inc`) ; ne rien bootstrapper pour les hooks token.

### 2026-05-16 — hook_token_info() dans hook_install() — jamais découvert

- **Symptôme :** Les tokens custom n'apparaissent pas dans le Token Tree Browser
- **Cause :** `hook_token_info()` déclaré dans `hook_install()` au lieu du `.module` — il ne persiste pas
- **Correct :** `hook_token_info()` et `hook_tokens()` DOIVENT être dans le fichier `.module` (ou `.tokens.inc`)
- **Prévention :** Ces hooks sont des hooks de découverte — toujours dans le fichier principal du module + `drush cr`
