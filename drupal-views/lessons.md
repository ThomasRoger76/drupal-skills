# Leçons — drupal-views

Bugs et pièges découverts en projet réel. Mis à jour après chaque incident.

---

## 2026-05-16 — Création du skill

### Template `views-fields--VIEW-ID.html.twig` ignoré silencieusement
- **Symptôme :** Le template Twig n'est jamais utilisé pour la View
- **Cause :** Le nom correct est `views-view-fields--VIEW-ID--DISPLAY-ID.html.twig` (avec `view-` avant `fields`)
- **Correct :** Toujours copier le nom depuis les commentaires Twig debug HTML — lire la suggestion exacte
- **Prévention :** Règle : debug → copier le nom complet → remplacer `__` par `--`

### `Views::getView()` retourne NULL sans erreur
- **Symptôme :** PHP Fatal "Call to a member function setDisplay() on null"
- **Cause :** La View a été supprimée ou renommée — `getView()` retourne NULL sans exception
- **Correct :** `if ($view = Views::getView('ma_view')) { ... }` — toujours vérifier avant d'appeler des méthodes
- **Prévention :** Pattern obligatoire : `$view = Views::getView(); if (!$view || !$view->access($display_id)) { return []; }`

### hook_views_data() — `drush cr` obligatoire après modification
- **Symptôme :** Le nouveau champ ou la nouvelle table n'apparaît pas dans l'UI Views
- **Cause :** Views met en cache la découverte des tables — le changement n'est pas pris en compte
- **Correct :** `drush cr` après chaque modification de hook_views_data()
- **Prévention :** Toujours inclure dans la procédure de dev Views : modifier → `drush cr` → vérifier dans l'UI

### Chargement d'entités dans `render()` du FieldHandler — N+1
- **Symptôme :** 50 nœuds dans une View = 51 requêtes (1 par entité référencée)
- **Cause :** `Entity::load($id)` dans la méthode `render()` est appelé pour chaque ligne
- **Correct :** Utiliser `preRender()` pour charger toutes les entités en batch avec `loadMultiple()`
- **Prévention :** Ne jamais charger des entités dans `render()` — utiliser `preRender()` + `$this->entityStorage->loadMultiple($ids)`

### Views AJAX — jQuery absent en D10/D11
- **Symptôme :** La soumission AJAX ne fonctionne plus après upgrade D9→D10
- **Cause :** jQuery n'est plus chargé par défaut — les scripts AJAX Drupal Views ont été réécrits en vanilla JS
- **Correct :** Vérifier que `core/jquery` n'est pas une dépendance du thème pour les Views AJAX
- **Prévention :** Ne pas dépendre de jQuery dans les handlers Views custom — utiliser vanilla JS

### Contextual filter absent → affichage de tous les résultats
- **Symptôme :** La View affiche tous les nœuds alors qu'elle devrait n'en montrer que quelques-uns
- **Cause :** "Provide default value" n'est pas configuré — si l'argument est absent, Views ignore le filtre
- **Correct :** Configurer "When the filter value is NOT available" → "Provide default value" (fixed: 0) ou "Display empty text"
- **Prévention :** Toujours tester la View sans argument dans l'URL — le comportement par défaut doit être explicite

### Access plugin "None" — données sensibles exposées
- **Symptôme :** Des données non publiées ou privées apparaissent dans la View
- **Cause :** Access plugin = "None" → aucune vérification de permissions
- **Correct :** Access plugin → "Permission" + sélectionner la permission appropriée
- **Prévention :** Règle : toute View avec des données non-publiques doit avoir un access plugin != "None"

### `$view->result` manipulé après `execute()` — résultats non rendus
- **Symptôme :** Modifications de `$view->result` dans un hook n'apparaissent pas dans le rendu
- **Cause :** Après `execute()`, le rendu est déjà en cours — modifier `result` directement est trop tard pour certains traitements
- **Correct :** Utiliser `hook_views_post_execute()` pour modifier les résultats avant le rendu
- **Prévention :** `hook_views_post_execute()` = modifier les données | `hook_views_pre_render()` = modifier les attachments/cache
