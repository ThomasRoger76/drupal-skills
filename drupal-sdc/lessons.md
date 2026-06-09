# Leçons — drupal-sdc

Erreurs SDC découvertes en projet Drupal 10.3+/D11.

---

### 2026-05-16 — Composant non découvert — nom de fichier incorrect

- **Symptôme :** `Component 'mon_theme:card' not found` dans Twig
- **Cause :** Le fichier `.twig` ne porte pas le même nom que le répertoire (`card/mon-template.html.twig` au lieu de `card/card.twig`)
- **Correct :** Le fichier Twig DOIT avoir le même nom que le répertoire : `card/card.twig`
- **Prévention :** Règle : `components/NOM/NOM.twig` — exactement le même nom partout

### 2026-05-16 — Props non typées — erreur silencieuse en dev

- **Symptôme :** Une prop `url` reçoit un objet `Url` Drupal au lieu d'une string — Twig rend un objet vide
- **Cause :** Props non typées dans `.component.yml` — aucune validation ni conversion
- **Correct :** Déclarer `url: { type: string, format: uri }` dans les props + convertir l'objet avant de passer la prop
- **Prévention :** Activer `sdc.enforce_schemas: true` dans services.yml en développement — la validation stricte prévient ces erreurs (le paramètre `sdc.debug` n'existe pas dans le core)

### 2026-05-16 — JS double-initialisation — Drupal.behaviors non utilisé

- **Symptôme :** Le comportement JavaScript du composant s'exécute plusieurs fois
- **Cause :** Code JS dans un `document.ready()` ou `window.onload` au lieu de `Drupal.behaviors`
- **Correct :** Pattern obligatoire : `Drupal.behaviors.monComposant = { attach: function(context) { ... } }`
- **Prévention :** Toujours utiliser `Drupal.behaviors` dans le JS SDC

### 2026-05-16 — `cache:rebuild` requis après création d'un composant

- **Symptôme :** Le nouveau composant SDC retourne une erreur "Component not found" même si les fichiers existent
- **Cause :** Le registre SDC est mis en cache — Drupal ne découvre pas les nouveaux composants automatiquement
- **Correct :** `drush cr` après chaque création ou renommage de composant SDC
- **Prévention :** En développement, ajouter `auto_reload: true` dans `services.yml` pour Twig + `drush cr` après création

### 2026-06-08 — Service SDC : `plugin.manager.sdc`, jamais `sdc.component_registry`

- **Symptôme :** `drush php:eval` de debug lève `You have requested a non-existent service "sdc.component_registry"`.
- **Cause :** Le service `sdc.component_registry` n'existe pas. Le registre SDC est la classe
  `ComponentPluginManager`, exposée sous l'ID `plugin.manager.sdc`.
- **Correct :** `\Drupal::service('plugin.manager.sdc')` → `getDefinitions()` (liste, clés = IDs)
  et `find('mon_theme:card')` (instance `Component`, avec `->metadata`).
- **Prévention :** Lister via `array_keys($manager->getDefinitions())`. La métadonnée props/slots
  est dans `$component->metadata->schema`.

### 2026-06-08 — Slot rendu via `{{ nom_slot }}`, PAS `{{ slots.nom_slot }}`

- **Symptôme :** Confusion sur la variable Twig à utiliser pour un slot.
- **Cause :** Croyance erronée qu'un slot s'accède via un objet `slots.*`. SDC injecte
  les slots ET les props comme variables Twig de premier niveau dans le contexte du
  composant (cf. `ComponentNodeVisitor` / `mergeAdditionalRenderContext` dans le core).
- **Correct :** Props ET slots s'utilisent pareil : `{{ title }}`, `{{ badge }}`, `{{ footer }}`.
  Côté appelant, un slot se remplit avec `{% embed %}` + `{% block badge %}…{% endblock %}`,
  et la prop avec `with { title: '...' }`.
- **Prévention :** Ne jamais écrire `{{ slots.badge }}` — la variable n'existe pas.
  La distinction props/slots est dans le `.component.yml` et la façon de PASSER la valeur,
  pas dans la façon de la LIRE en Twig.

### 2026-05-16 — CSS SDC non chargé — fichier mal nommé

- **Symptôme :** Les styles du composant ne s'appliquent pas
- **Cause :** Le fichier CSS s'appelle `styles.css` au lieu de `card.css` — doit avoir le même nom que le répertoire
- **Correct :** Renommer en `card.css` (même nom que le répertoire, le `.twig`, et le `.component.yml`)
- **Prévention :** Convention SDC stricte : `components/card/card.css`, `card/card.twig`, `card/card.component.yml`

### 2026-05-16 — Props validation silencieuse en prod — bug invisible

- **Symptôme :** Une prop avec mauvais type dégrade l'affichage sans erreur visible
- **Cause :** `sdc.enforce_schemas: false` (défaut en production) désactive la validation stricte des props
- **Correct :** Écrire des tests Functional qui vérifient le rendu SDC avec données invalides
- **Prévention :** `sdc.enforce_schemas: true` en dev (`services.yml`), tests PHPUnit pour les cas limites en prod
