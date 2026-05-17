# Changelog — drupal-performance

---

## v1.0 — 2026-05-16

**Création initiale**

### Couverture

**`SKILL.md`**
- Architecture des couches de cache (6 niveaux : Varnish → Page Cache → DPC → BigPipe → Render Cache → DB)
- Quick Decision Table (26+ entrées)
- Anti-patterns critiques (10 entrées)
- Table versioning D8→D11 (BigPipe D8.2, JIT PHP8.3)

**`cache-system.md`**
- Page Cache — configuration, exclusions, invalidation
- Dynamic Page Cache — render arrays avec #cache, contexts corrects
- BigPipe — `#lazy_builder`, `TrustedCallbackInterface`, configuration
- Render Cache — `#cache` complet (tags, contexts, max-age)
- Cache bins Drupal (default, render, data, config...)
- Désactiver le cache en développement (settings.local.php complet)
- Configuration des backends (settings.php)
- Debug du cache (headers curl, drush)

**`cache-tags-contexts.md`**
- Explication conceptuelle Tags/Contexts/Max-age
- Tags d'entités natifs Drupal (node:42, node_list, config:system.site...)
- Obtenir les tags d'une entité avec `getCacheTags()`
- Cache::invalidateTags() — usage et timing
- Contexts natifs Drupal (url, user, user.roles, languages...)
- Choisir le context le plus précis
- Cache context custom (classe + service)
- Patterns complets par cas d'usage (nœud, bloc, liste, API externe)
- Debugging

**`database-optimization.md`**
- Slow query log MariaDB (configuration Docker)
- EXPLAIN — interprétation
- N+1 queries — identification et 3 solutions (loadMultiple, Views JOIN, Database API)
- N+1 avec Paragraphs — pattern batch loading
- hook_schema() avec indexes complets
- EntityQuery optimisée (accessCheck, count, range)
- Compter les requêtes avec Devel
- Checklist optimisation DB

**`lessons.md`**
- 8 incidents performance résolus avec diagnostics et corrections

---

## Compatibilité Drupal

| Skill version | Drupal | Notes |
|--------------|--------|-------|
| v1.0 | D8, D9, D10, D11 | BigPipe D8.2+, JIT PHP 8.1+, OPcache preload PHP 7.4+ |
