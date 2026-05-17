# Changelog — drupal-api

---

## v1.0 — 2026-05-16

**Création initiale**

### Couverture

**`SKILL.md`**
- Tableau comparatif JSON:API vs REST API vs GraphQL (5 critères)
- Quick Decision Table (25+ entrées)
- Anti-patterns critiques (10 entrées)
- Table versioning D8→D11 (JSON:API core D8.7, drupal/next D9+)

**`jsonapi-deep.md`**
- Référence des endpoints (GET/POST/PATCH/DELETE)
- Filtres simples et avec opérateurs (15 opérateurs disponibles)
- Groupes de filtres AND/OR imbriqués
- Includes (relations eager loaded)
- Sparse fieldsets (limiter les champs retournés)
- Pagination (offset-based, liens de navigation)
- Tri (simple, multiple, sur relations)
- Mutations POST/PATCH/DELETE avec headers requis
- Obtenir le X-CSRF-Token
- Sécurisation (`hook_jsonapi_entity_filter_access()`)
- Masquer des champs (jsonapi_extras)
- Cache JSON:API (X-Drupal-Cache-Tags headers)
- Exemples JavaScript (fetch API, GET et POST)

**`authentication.md`**
- Installation Simple OAuth + génération des clés RSA
- Grant types — tableau comparatif (client_credentials, auth_code+PKCE, refresh)
- Client Credentials — créer un Consumer, obtenir un token (curl + JS)
- Authorization Code + PKCE — flow complet côté frontend (code_verifier, code_challenge, callback)
- Refresh Token — renouveler sans reconnexion
- Configuration CORS (services.yml)
- Sécuriser les routes API (check du token OAuth dans Controller)

**`lessons.md`**
- 7 incidents headless réels avec corrections

---

## Compatibilité Drupal

| Skill version | Drupal | Notes |
|--------------|--------|-------|
| v1.0 | D8, D9, D10, D11 | JSON:API core D8.7+, Simple OAuth contrib, drupal/next contrib D9+ |
