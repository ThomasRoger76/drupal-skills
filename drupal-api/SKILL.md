---
name: drupal-api
description: Use when building headless or decoupled Drupal with JSON:API (filtering, includes, sparse fieldsets, pagination, mutations), setting up Simple OAuth for API authentication (client_credentials, authorization_code), configuring CORS for frontend applications, integrating with Next.js via the next-drupal module (preview mode, ISR, revalidation), implementing GraphQL with the graphql module, creating custom REST API resources (RestResource plugin), securing API endpoints with hook_jsonapi_entity_filter_access, setting up webhook notifications for content changes, building a Drupal-powered mobile API, or configuring JSON:API extras for field aliasing and endpoint customization in Drupal 8-11+
---

# Drupal API / Headless — Référence Complète

## Overview

Référentiel complet de Drupal en mode API-first / headless 8-11+ : JSON:API (core), Simple OAuth, CORS, Next.js + Drupal, GraphQL, REST Resources custom, webhooks pour invalidation de cache frontend, sécurisation des endpoints. Drupal est le CMS headless le plus puissant grâce à JSON:API en core depuis D8.7.

## 🎯 La Règle Fondamentale

> **JSON:API first.** Avant de créer un endpoint custom (RestResource), vérifier que JSON:API ne suffit pas avec les filtres appropriés. JSON:API gère automatiquement la pagination, les includes, les permissions Drupal, et le cache.

---

## JSON:API vs REST API vs GraphQL — Tableau Décisionnel

| Critère | JSON:API (core) | REST API (core) | GraphQL (contrib) |
|---------|----------------|-----------------|-------------------|
| Disponibilité | Core D8.7+ | Core D8 | Contrib |
| Requête flexible (champs à la demande) | ✅ sparse fieldsets | ❌ | ✅ |
| Filtrage avancé | ✅ filter[field][value] | ❌ limité | ✅ |
| Includes (eager loading) | ✅ ?include=field | ❌ | ✅ |
| Mutations (POST/PATCH/DELETE) | ✅ | ✅ | ✅ |
| Cache Drupal natif | ✅ | ✅ | ⚠️ |
| Authentification | OAuth / Cookie / Basic | OAuth / Cookie / Basic | OAuth / Cookie |
| Recommandé pour | Consumer général | Endpoint spécifique | Frontend complexe |

**Règle :** utiliser JSON:API par défaut. REST Resource custom uniquement si besoin de logique métier non exposable via JSON:API. GraphQL pour les frontends avec besoins de requêtes très complexes.

---

## Quick Decision Table

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Lire des nœuds en JSON depuis un frontend | `GET /jsonapi/node/article` | [jsonapi-deep.md](jsonapi-deep.md) |
| Filtrer par champ (ex: status=published) | `?filter[status]=1` | [jsonapi-deep.md](jsonapi-deep.md) |
| Filtrer avec opérateur (CONTAINS, IN, >=) | `?filter[title][operator]=CONTAINS&filter[title][value]=foo` | [jsonapi-deep.md](jsonapi-deep.md) |
| Filtres combinés (AND / OR) | filter groups avec conjunction | [jsonapi-deep.md](jsonapi-deep.md) |
| Inclure les entités référencées (eager load) | `?include=field_tags,field_image` | [jsonapi-deep.md](jsonapi-deep.md) |
| Limiter les champs retournés (sparse fieldsets) | `?fields[node--article]=title,body` | [jsonapi-deep.md](jsonapi-deep.md) |
| Pagination JSON:API | `?page[limit]=10&page[offset]=20` | [jsonapi-deep.md](jsonapi-deep.md) |
| Trier les résultats | `?sort=-created,title` | [jsonapi-deep.md](jsonapi-deep.md) |
| Créer/modifier un nœud via JSON:API | `POST/PATCH /jsonapi/node/article` + token CSRF | [jsonapi-deep.md](jsonapi-deep.md) |
| Authentification machine-to-machine | Simple OAuth — client_credentials grant | [authentication.md](authentication.md) |
| Authentification utilisateur frontend | Simple OAuth — authorization_code + PKCE | [authentication.md](authentication.md) |
| Token JWT avec Simple OAuth | `drupal/simple_oauth` + `drupal/jwt` | [authentication.md](authentication.md) |
| Configurer CORS pour un frontend | `services.yml` → `cors.config:` | [cors-webhooks.md](cors-webhooks.md) |
| Sécuriser l'accès à un bundle JSON:API | `hook_jsonapi_entity_filter_access()` | [jsonapi-deep.md](jsonapi-deep.md) |
| Masquer des champs de JSON:API | `drupal/jsonapi_extras` — field override | [jsonapi-deep.md](jsonapi-deep.md) |
| Aliaser les endpoints JSON:API | `drupal/jsonapi_extras` — resource config | [jsonapi-deep.md](jsonapi-deep.md) |
| Frontend Next.js + Drupal | Module `drupal/next` + `next-drupal` npm | [nextjs-drupal.md](nextjs-drupal.md) |
| Preview de brouillons dans Next.js | Next.js Preview Mode + Simple OAuth | [nextjs-drupal.md](nextjs-drupal.md) |
| Revalidation ISR après modification | Webhook Drupal → Next.js `revalidatePath` | [nextjs-drupal.md](nextjs-drupal.md) |
| GraphQL — requêtes flexibles | Module `drupal/graphql` v4 | [graphql.md](graphql.md) |
| Webhook sur modification de contenu | `hook_entity_presave` / EventSubscriber + HTTP call | [cors-webhooks.md](cors-webhooks.md) |
| Endpoint REST custom avec logique métier | `RestResource` plugin custom | [cors-webhooks.md](cors-webhooks.md) |
| JSON:API pour contenu multilingue | `?filter[langcode]=fr` + `drupal-langcode` header | [jsonapi-deep.md](jsonapi-deep.md) |
| Performances JSON:API (cache) | `X-Drupal-Cache-Tags` headers + Varnish | [jsonapi-deep.md](jsonapi-deep.md) |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| JSON:API sans authentification sur mutations | Toujours `simple_oauth` ou session cookie pour POST/PATCH/DELETE | N'importe qui peut modifier le contenu |
| CORS `allowedOrigins: ['*']` en production | Whitelist des domaines frontend autorisés | Sécurité CORS inexistante |
| Exposer `/jsonapi` sans permissions Drupal | Configurer les permissions par bundle | Données sensibles accessibles anonymement |
| Token OAuth hardcodé dans le code frontend | Variables d'environnement + refresh token | Fuite de credentials |
| REST Resource custom pour ce que JSON:API fait | Utiliser JSON:API avec filtres appropriés | Maintenance inutile |
| `GET /jsonapi/node/article` sans `page[limit]` | Toujours paginer — limiter à 50 max | Réponse de plusieurs MB, timeout |
| JSON:API sans `drupal/jsonapi_extras` pour les champs sensibles | Masquer les champs password, tokens | Exposition de données internes |
| Webhook sans validation de signature | HMAC-SHA256 sur le payload | N'importe qui peut déclencher le webhook |
| Next.js sans Preview Mode pour les brouillons | Configurer `drupal/next` avec preview mode | Les éditeurs ne voient pas leurs modifications |
| GraphQL sans persisted queries en prod | Activer persisted queries + désactiver introspection | Schema exposé + queries arbitraires |

## Évolution par Version Majeure

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| JSON:API (core) | contrib | ✅ core D8.7+ | ✅ | ✅ |
| REST API (core) | ✅ | ✅ | ✅ | ✅ |
| Simple OAuth (contrib) | ✅ | ✅ | ✅ | ✅ |
| GraphQL v4 (contrib) | ✅ | ✅ | ✅ | ✅ |
| `drupal/next` module | ❌ | ✅ | ✅ | ✅ |
| JSON:API mutations (POST/PATCH/DELETE) | ✅ | ✅ | ✅ | ✅ |
| `drupal/jsonapi_extras` | ✅ | ✅ | ✅ | ✅ |
| CORS core config | ✅ | ✅ | ✅ | ✅ |
| Decoupled Router (contrib) | ✅ | ✅ | ✅ | ✅ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Problèmes rencontrés en projets headless réels.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions.

## See Also

- `drupal-security` — CSRF tokens pour mutations JSON:API, OAuth2, CORS
- `drupal-core` — `RestResource` plugin, JSON:API accès, permissions
- `drupal-multilingual` — JSON:API avec contenu traduit, `Accept-Language` header
- `drupal-performance` — Cache JSON:API, `X-Drupal-Cache-Tags`, Varnish
- `drupal-config` — Configuration JSON:API extras, Simple OAuth settings
- `drupal-migration` — Importer du contenu via JSON:API, webhooks post-migration
