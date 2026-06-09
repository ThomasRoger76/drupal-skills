# Changelog — drupal-multilingual

---

## v1.1 — 2026-06-09

**Audit qualité — corrections de fond + alignement standards**

### Corrigé
- **Config de négociation de langue inexacte** — remplacement du fichier fantôme
  `language.negotiation.yml` par les vrais fichiers core `language.types.yml`
  (`configurable` + `negotiation.<type>.enabled` / `method_weights`, méthodes
  `language-url`/`language-session`/...) et `language.negotiation.url.yml`
  (`source: path_prefix|domain`). Corrigé dans `multilingual-setup.md` et
  `language-negotiation.md`.
- **Docker natif** — toutes les commandes `drush` exécutables passent désormais par
  `docker compose exec php drush ...` (jamais `ddev`, jamais `drush` nu) sur les 7
  fichiers. Note de convention ajoutée dans `SKILL.md` et `multilingual-setup.md`.
- **TMGMT** — `$job->translator = ...` remplacé par `$job->set('translator', ...)`
  (API entité correcte) dans `tmgmt.md`.
- **Module geolocation fantaisiste** — `drupal/geoip_language` (inexistant) remplacé
  par la combinaison réelle `drupal/smart_ip` + `drupal/geoip` dans `SKILL.md`.
- `config-translation.md` — `var_dump` remplacé par `var_export` dans le snippet de
  vérification (pas de débug brut).

### Ajouté
- `content-translation.md` — section « Chercher les entités AYANT une traduction »
  avec le piège `langcode` vs `default_langcode` (clarifie le bug n°7 des lessons).
- `lessons.md` — 3 nouvelles leçons (config négociation, Docker, TMGMT set()).

---

## v1.0 — 2026-05-16

**Création initiale**

### Couverture

**`SKILL.md`**
- Architecture des 4 modules core multilinguisme
- Quick Decision Table (25+ entrées)
- Anti-patterns critiques (10 entrées)
- Table versioning D8→D11 (JSON:API multilingue amélioré D10)

**`multilingual-setup.md`**
- Les 4 modules core dans l'ordre d'activation
- Ajout de langues (drush + YAML config)
- Configuration négociation de langue (URL prefix, domain)
- Language Fallback (YAML config)
- CSS Logical Properties pour RTL (tableau complet équivalences)
- Pathauto par langue
- Bloc Language Switcher (config YAML)
- Commandes Drush (language:add, locale:import, locale:update...)
- Checklist post-setup

**`content-translation.md`**
- Activer la traduction d'un Content Type (YAML + UI)
- Créer une traduction programmatiquement (`addTranslation`)
- Mettre à jour une traduction existante (`getTranslation()` + save)
- Charger la traduction courante (`getTranslationFromContext`)
- Lister les traductions d'un nœud
- Supprimer une traduction
- EntityQuery multilingue (attention langcode vs traductions)
- Générer des URLs par langue (`Url::fromRoute` avec language)
- Pattern preprocess correct (getTranslationFromContext + cache context)
- Pattern preprocess incorrect et pourquoi

**`lessons.md`**
- 7 bugs multilingues réels avec corrections précises

---

## Compatibilité Drupal

| Skill version | Drupal | Notes |
|--------------|--------|-------|
| v1.0 | D8, D9, D10, D11 | TMGMT contrib, Pathauto contrib, JSON:API multilingue amélioré D10+ |
| v1.1 | D8, D9, D10, D11 | Config négociation corrigée (language.types.yml / language.negotiation.url.yml), Docker natif, TMGMT set() |
