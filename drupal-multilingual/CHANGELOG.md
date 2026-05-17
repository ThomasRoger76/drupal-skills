# Changelog — drupal-multilingual

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
