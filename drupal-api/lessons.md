# Leçons — drupal-api

Incidents en projets headless/decoupled réels. Mis à jour après chaque résolution.

---

## 2026-05-16 — Création du skill

### JSON:API sans `page[limit]` → réponse de 10MB et timeout
- **Symptôme :** `GET /jsonapi/node/article` timeout après 30 secondes en production
- **Cause :** Sans `page[limit]`, Drupal retourne TOUTES les entités — un site avec 5000 articles génère une réponse de plusieurs MB
- **Correct :** Toujours paginer : `?page[limit]=20&page[offset]=0`
- **Prévention :** Configurer la limite max JSON:API dans la config du module ou le code : maximum 50 items par page

### CORS — `Access-Control-Allow-Origin` absent → frontend bloqué
- **Symptôme :** Les requêtes du frontend Next.js sont bloquées avec "CORS policy" dans la console
- **Cause :** La config CORS dans `services.yml` n'était pas correctement rechargée après modification
- **Correct :** Modifier `services.yml` → `drush cr` → vérifier avec `curl -H "Origin: https://frontend.com" https://drupal.com/jsonapi/node/article`
- **Prévention :** Tester CORS depuis le navigateur ET depuis curl. `drush cr` est obligatoire après modification de services.yml

### X-CSRF-Token expiré → mutation JSON:API 403
- **Symptôme :** POST JSON:API fonctionne pendant 1 heure puis retourne 403
- **Cause :** Le X-CSRF-Token Drupal expire avec la session. Le frontend gardait l'ancien token.
- **Correct :** Re-fetch le token avant chaque mutation importante, ou implémenter un refresh automatique
- **Prévention :** Pattern : récupérer `/session/token` avant chaque POST/PATCH/DELETE — ne pas cacher le token côté client

### Simple OAuth — clés RSA non générées → erreur 500 sur /oauth/token
- **Symptôme :** `/oauth/token` retourne HTTP 500 — log : "Failed to sign with private key"
- **Cause :** Les clés RSA n'ont jamais été générées après installation de Simple OAuth
- **Correct :** `openssl genrsa -out private.key 2048 && openssl rsa -in private.key -pubout -out public.key` puis configurer les chemins dans la config Simple OAuth
- **Prévention :** La génération des clés RSA est une étape POST-installation obligatoire — l'inclure dans le README du projet

### Consumer configuré en non-confidential → client_credentials impossible
- **Symptôme :** `grant_type=client_credentials` retourne "Client authentication failed"
- **Cause :** Le Consumer doit être marqué "Confidential" pour utiliser client_credentials
- **Correct :** `/admin/config/services/consumer/{id}/edit` → cocher "Is this client confidential?"
- **Prévention :** client_credentials = client confidentiel. authorization_code = peut être public (SPA)

### Next.js — preview mode non fonctionnel pour les brouillons
- **Symptôme :** Les éditeurs voient les brouillons en production mais pas dans le preview Next.js
- **Cause :** Le module `drupal/next` n'était pas installé — le preview mode utilisait un token invalide
- **Correct :** `composer require drupal/next && drush en next next_jsonapi -y` + configurer les sites Next dans Drupal
- **Prévention :** L'intégration Next.js+Drupal nécessite les modules côté Drupal ET le package npm `next-drupal` côté frontend

### JSON:API — champs sensibles exposés (mots de passe hashés, tokens)
- **Symptôme :** L'endpoint `GET /jsonapi/user/user` expose des données internes
- **Cause :** JSON:API expose par défaut tous les champs d'une entité auxquels l'utilisateur a accès
- **Correct :** `drupal/jsonapi_extras` → désactiver les champs sensibles (pass, login, access, field_internal_token)
- **Prévention :** Toujours auditer les champs exposés par JSON:API avant la mise en production — installer jsonapi_extras dès le début
