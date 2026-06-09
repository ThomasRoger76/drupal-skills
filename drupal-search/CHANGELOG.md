# Changelog — drupal-search

---

## v1.1 — 2026-06-09

**Audit qualité — corrections de défauts factuels et standards D11**

### Corrigé
- **`custom-processors.md`** — Ajout d'un avertissement en tête : Search API conserve l'annotation Doctrine `@SearchApiProcessor` (PAS d'attribut PHP `#[]`, classe `Attribute` inexistante). Évite la conversion erronée « D11 = attributs ».
- **`elasticsearch.md`** — `elasticsearch_connector` et `search_api_opensearch` présentés comme modules concurrents (tableau de choix) au lieu d'être installés ensemble. YAML serveur corrigé pour `elasticsearch_connector` 8.x (connexion portée par le serveur, plus d'entité « cluster »). Référence `drush solr-gsc` obsolète retirée.
- **`solr-integration.md`** — URLs `curl` quotées (RELOAD du core, `select`) — le `&` non échappé lançait un job shell. Faute de frappe `preprocress_index` → `preprocess_index` dans le YAML highlight.
- **`search-api-setup.md`** — Settings inventés `role_field`/`user_field` du processor `content_access` retirés (config réelle `{}`). Hook `entity_update` réécrit : suppression du code mort (`$items` inutilisé), tracking IDs corrects avec langcode, `use` ajoutés, early-return.
- **`SKILL.md`** — Quick Decision Table : entrée annotation vs attribut. OpenSearch : mention des deux modules. See Also : renvoi vers le skill `solr` autonome.

### Vérifié (sans changement)
- Annotations `@SearchApiProcessor` confirmées correctes pour D8-D11 (module non migré aux attributs).

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
| v1.1 | D9, D10, D11 | Idem v1.0 — processors custom en annotation `@SearchApiProcessor` (module non migré aux attributs) |
