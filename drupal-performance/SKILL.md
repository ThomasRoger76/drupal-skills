---
name: drupal-performance
description: Use when optimizing Drupal site performance, configuring render cache with tags/contexts/max-age, enabling BigPipe for personalized content, setting up Dynamic Page Cache, configuring Redis or Memcache as cache backends, integrating Varnish with the Purge module and cache-tags HTTP header, debugging N+1 queries on entity references, optimizing PHP OPcache and JIT, configuring CSS/JS aggregation, profiling slow pages with XHProf or Blackfire, fixing Core Web Vitals (LCP, CLS, INP), preventing cache stampede, configuring CDN cache busting, or analyzing slow Views queries with EXPLAIN in Drupal 8-11+
---

# Drupal Performance — Référence Complète

## Overview

Référentiel complet de l'optimisation des performances Drupal 8-11+ : système de cache (Page Cache, Dynamic Page Cache, Render Cache, BigPipe), reverse proxy (Varnish, CDN), backends de cache (Redis, Memcache), base de données (N+1, indexes, slow query log), PHP (OPcache, JIT, APCu), frontend (agrégation CSS/JS, Core Web Vitals), profilage (XHProf, Blackfire, Devel).

## 🎯 La Règle Fondamentale

> **Cache granulaire, pas cache nucléaire.** `drush cache:rebuild` est un outil de debug, pas une solution de prod. Chaque page lente a une cause précise. Diagnostiquer d'abord, configurer le cache ensuite.

---

## Couches de Cache — Architecture Drupal

```
Requête HTTP entrante
       ↓
  [1] Varnish / CDN (reverse proxy — avant PHP)
       ↓ (si MISS)
  [2] Page Cache (Drupal — utilisateurs anonymes, réponse HTTP complète)
       ↓ (si utilisateur connecté)
  [3] Dynamic Page Cache (Drupal — parties variables mises en cache séparément)
       ↓
  [4] BigPipe (streaming — blocs personnalisés chargés en SSE après le HTML)
       ↓
  [5] Render Cache (composants individuels — blocs, entités, render arrays)
       ↓
  [6] Database / Entity Cache (données brutes)
```

**Règle :** toujours diagnostiquer à quelle couche se situe le problème avant d'intervenir.

---

## Quick Decision Table

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Pages lentes pour visiteurs anonymes | Page Cache (module core, activé par défaut) | [cache-system.md](cache-system.md) |
| Pages lentes pour utilisateurs connectés | Dynamic Page Cache (module core) | [cache-system.md](cache-system.md) |
| Blocs personnalisés bloquent le rendu | BigPipe — lazy placeholders SSE | [cache-system.md](cache-system.md) |
| Contenu périmé malgré édition | Cache tags manquants sur le render array | [cache-tags-contexts.md](cache-tags-contexts.md) |
| Cache invalidé trop souvent | Cache contexts trop larges (url vs route) | [cache-tags-contexts.md](cache-tags-contexts.md) |
| Bloquer le cache pour une partie dynamique | `'max-age' => 0` sur le render array ciblé | [cache-tags-contexts.md](cache-tags-contexts.md) |
| Redis comme backend de cache principal | `drupal/redis` + settings.php config | [redis-memcache.md](redis-memcache.md) |
| Memcache comme backend | `drupal/memcache` module + settings.php | [redis-memcache.md](redis-memcache.md) |
| Reverse proxy Varnish devant Drupal | Purge module + varnish_purger + VCL Drupal | [varnish-cdn.md](varnish-cdn.md) |
| Invalider Varnish à la sauvegarde d'un nœud | Cache-Tags HTTP header + Purge module | [varnish-cdn.md](varnish-cdn.md) |
| CDN pour les assets statiques (images, CSS) | CDN module ou configuration webserver | [varnish-cdn.md](varnish-cdn.md) |
| Cache busting après déploiement CSS/JS | `drush cr` + Varnish purge all | [varnish-cdn.md](varnish-cdn.md) |
| Requêtes SQL lentes (slow query log) | MariaDB slow_query_log + EXPLAIN | [database-optimization.md](database-optimization.md) |
| N+1 queries sur les entity references | EntityQuery preloading ou Views avec JOIN | [database-optimization.md](database-optimization.md) |
| Index manquant sur table custom | `hook_schema` avec `indexes:` | [database-optimization.md](database-optimization.md) |
| OPcache non configuré en production | opcache.ini : memory, files, revalidate | [php-opcache.md](php-opcache.md) |
| JIT PHP 8.3 pour améliorer les perfs | `opcache.jit = 1255` — benchmarker avant | [php-opcache.md](php-opcache.md) |
| APCu pour le cache user space | `drupal/apcu_serializer` + extension APCu | [php-opcache.md](php-opcache.md) |
| CSS/JS non agrégé en production | `system.performance` css.preprocess + js.preprocess | [frontend-performance.md](frontend-performance.md) |
| Core Web Vitals : LCP trop lent | BigPipe + `loading="lazy"` sur images off-screen | [frontend-performance.md](frontend-performance.md) |
| Core Web Vitals : CLS (layout shift) | Dimensions images dans HTML, font-display:swap | [frontend-performance.md](frontend-performance.md) |
| Core Web Vitals : INP (interaction delay) | Réduire JS bloquant, `once()` pattern correct | [frontend-performance.md](frontend-performance.md) |
| Profiler une page lente | Devel + Kint / XHProf / Blackfire.io | [profiling.md](profiling.md) |
| Analyser les requêtes d'une page | Devel — Query Log + `dpq()` | [profiling.md](profiling.md) |
| Désactiver tout le cache en développement | settings.local.php null backends + Twig debug | [cache-system.md](cache-system.md) |
| Cache buster programmable (CI/CD) | `drush deploy` + Purge tag | [varnish-cdn.md](varnish-cdn.md) |
| Preload PHP pour réduire le bootstrap | `opcache.preload` avec fichier preload Drupal | [php-opcache.md](php-opcache.md) |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| `drush cr` comme solution à tout problème | Identifier la cause racine + cache tags précis | Résout rien, masque le vrai problème |
| `#cache['max-age'] => 0` globalement | Cibler uniquement les composants vraiment dynamiques | Désactive toute mise en cache |
| `\Drupal::cache()->deleteAll()` en prod | Purge module + cache tags ciblés | Efface tout le cache simultanément pour tous |
| Render array sans `#cache` sur contenu dynamique | Toujours définir tags + contexts + max-age | Contenu périmé ou sur-invalidé |
| `opcache.validate_timestamps = 1` en prod | `validate_timestamps = 0` + `drush cr` après déploiement | Relecture fichiers PHP à chaque requête |
| Varnish sans module Purge | Purge + varnish_purger + `X-Drupal-Cache-Tags` | Cache Varnish jamais invalidé |
| Entity reference chargée en boucle foreach | `$storage->loadMultiple($ids)` une seule fois | N+1 queries × nombre d'entités |
| BigPipe désactivé (module core) | Laisser BigPipe actif — accélère la perception | Blocs personnalisés bloquent tout le rendu |
| Cache contexts `url` au lieu de `url.path` | Utiliser le contexte le plus précis | Cache explosé inutilement |
| `\Drupal::cache('render')->deleteAll()` | Purge via tags : `Cache::invalidateTags(['node:42'])` | Wipe complet du cache render |

## Évolution par Version Majeure

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| Page Cache (core) | ✅ | ✅ | ✅ | ✅ |
| Dynamic Page Cache (core) | ✅ D8.2+ | ✅ | ✅ | ✅ |
| BigPipe (core) | ✅ D8.2+ | ✅ | ✅ | ✅ |
| Cache tags + contexts | ✅ | ✅ | ✅ | ✅ |
| Redis module stable | contrib | contrib | contrib | contrib |
| Purge module (Varnish) | contrib | contrib | contrib | contrib |
| PHP JIT | ❌ | ❌ | ✅ PHP 8.1+ | ✅ PHP 8.3 |
| OPcache preload | ❌ | ✅ PHP 7.4+ | ✅ | ✅ |
| APCu serializer | contrib | contrib | contrib | contrib |
| `drush deploy` (cache rebuild inclus) | ❌ | ✅ | ✅ | ✅ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Problèmes de performances résolus en production.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions.

## See Also

- `drupal-docker` — OPcache en production, Redis/Memcache comme services Docker
- `drupal-core` — Cache tags dans les render arrays, `#cache` API
- `drupal-views` — Optimisation du cache des Views, N+1 sur les entity references
- `drupal-config` — Configuration de l'agrégation CSS/JS via system.performance
- `drupal-theming` — Lazy loading images, CSS critique, font-display
