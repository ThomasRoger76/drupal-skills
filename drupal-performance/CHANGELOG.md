# Changelog — drupal-performance

---

## v1.1 — 2026-06-09

**Corrections techniques (audit qualité)**

- **`php-opcache.md`** — Réécriture de la section OPcache Preload. L'ancien exemple bouclait sur
  tout `vendor/` (casse PHP-FPM au démarrage). Ajout d'un avertissement clair : Drupal ne supporte
  pas officiellement le preload (issue #3055735). Pattern correct curé + tolérant aux erreurs.
  Correction de la mention « preload_user obligatoire » (requis seulement si PHP-FPM démarre en root)
  et de la checklist.
- **`redis-memcache.md`** — Correction du bug `getenv('APP_ENV', 'prod')` : `getenv()` PHP n'accepte
  pas de 2e argument → remplacé par `(getenv('APP_ENV') ?: 'prod')`.
- **`cache-system.md`** — Pattern d'exclusion Page Cache complété avec l'API native (`CacheableMetadata`,
  `#cache['max-age'] => 0`). Correction du commentaire trompeur sur le context `user` et le DPC
  (le DPC met bien en cache une variante par utilisateur ; c'est `max-age 0` qui exclut).
- **`varnish-cdn.md`** — Correction du YAML `cdn.settings.yml` qui dupliquait la clé `type`
  (`simple` + `auto-balanced` dans le même mapping → invalide). Séparé en deux variantes.

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
| v1.1 | D8, D9, D10, D11 | Idem + corrections preload, getenv, DPC, CDN YAML |
