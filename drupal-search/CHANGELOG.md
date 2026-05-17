# Changelog — drupal-search

---

## v1.0 — 2026-05-16

**Création initiale**

### Couverture

**`SKILL.md`**
- Tableau comparatif backends (Database vs Solr vs Elasticsearch)
- Quick Decision Table (25+ entrées)
- Anti-patterns critiques (10 entrées)
- Table versioning D8→D11 (Search API Solr 4.x D10+)

**`search-api-setup.md`**
- Installation (composer + drush)
- Configuration serveur Database (YAML complet)
- Configuration serveur Solr (YAML complet avec connector)
- Création d'un index (YAML complet avec fieldSettings, datasource, processors)
- Processeurs disponibles — tableau de référence (12 processeurs)
- Commandes Drush (sapi-i, sapi-c, sapi-s, sapi-r, solr-gsc)
- Intégration Views + Search API (instructions)
- Indexation automatique et cron (configuration + hook pour réindexer relations)

**`facets.md`**
- Installation (facets + facets_summary)
- Créer une facette (YAML complet)
- Widgets disponibles (tableau 7 widgets)
- Processeurs — référence (8 processeurs essentiels)
- Placer les facettes comme blocs (YAML config block)
- Template Twig pour facettes (surcharge)
- Facets Summary — résumé des filtres actifs
- API PHP (DefaultFacetManager pour accès programmatique)
- Debug des facettes
- Checklist post-configuration

**`lessons.md`**
- 8 problèmes Search API/Facets réels avec corrections

---

## Compatibilité Drupal

| Skill version | Drupal | Notes |
|--------------|--------|-------|
| v1.0 | D9, D10, D11 | Search API contrib, Facets contrib, Search API Solr 4.x requis D10+ |
