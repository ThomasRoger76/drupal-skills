---
name: drupal-api — authentication
description: Authentification pour les APIs Drupal headless - Simple OAuth (client_credentials, authorization_code + PKCE), configuration des clients OAuth, rotation des tokens, et sécurisation des endpoints.
---

# Authentification API — Simple OAuth & JWT

## Installation Simple OAuth

```bash
composer require drupal/simple_oauth
drush en simple_oauth -y

# Générer les clés RSA (OBLIGATOIRE)
drush php:eval "
  \$config = \Drupal::configFactory()->getEditable('simple_oauth.settings');
  \$key_dir = \Drupal::root() . '/../oauth-keys/';
  if (!is_dir(\$key_dir)) { mkdir(\$key_dir, 0700); }

  // Générer les clés
  \$private_key = \$key_dir . 'private.key';
  \$public_key  = \$key_dir . 'public.key';

  exec(\"openssl genrsa -out \$private_key 2048\");
  exec(\"openssl rsa -in \$private_key -pubout -out \$public_key\");
  chmod(\$private_key, 0600);

  \$config->set('private_key', \$private_key)
         ->set('public_key', \$public_key)
         ->set('access_token_expiration', 3600)    // 1 heure
         ->set('refresh_token_expiration', 1209600) // 14 jours
         ->save();

  echo 'Clés générées dans ' . \$key_dir . PHP_EOL;
"
```

**⚠️ Mettre le répertoire des clés hors du webroot et dans `.gitignore`.**

---

## Grant Types — Choisir le Bon

| Grant Type | Cas d'usage | Credentials |
|-----------|-------------|-------------|
| `client_credentials` | Machine-to-machine (cron, microservices) | client_id + client_secret |
| `authorization_code + PKCE` | Applications frontend (Next.js, SPA) | Utilisateur se connecte via UI |
| `password` | **DÉPRÉCIÉ** — ne pas utiliser | username + password |
| `refresh_token` | Renouveler un access token | refresh_token |

---

## Client Credentials — Machine-to-Machine

### Créer un Client OAuth

```yaml
# config/install/consumer.entity.mon_service_api.yml
langcode: fr
status: true
uuid: 'uuid-du-consumer'
user_id: 1                    # Utilisateur associé (ses permissions sont utilisées)
label: 'Service Externe API'
description: 'Accès API pour le service de synchronisation.'
is_default: false
secret: '$2y$10$...'          # Hash bcrypt du secret
confidential: true            # true = client confidentiel (peut garder un secret)
redirect: ''                  # Pas de redirect pour client_credentials
roles:
  - api_consumer_role         # Rôle Drupal attribué au token
```

```bash
# Via l'UI : /admin/config/services/consumer/add
# Créer un "Consumer" avec :
# - Label : Service Externe API
# - Confidential : ✅ (client_credentials nécessite ça)
# - Roles : api_consumer_role (rôle avec les permissions nécessaires)
# - Générer un Client Secret aléatoire et le noter — affiché une seule fois
```

### Obtenir un Token

```bash
# Requête pour obtenir un access token
curl -X POST https://mon-site.com/oauth/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=UUID_DU_CONSUMER" \
  -d "client_secret=MON_SECRET"

# Réponse :
{
  "token_type": "Bearer",
  "expires_in": 3600,
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9...",
  "refresh_token": "def502..."
}
```

```javascript
// JavaScript — obtenir et utiliser le token
const tokenResponse = await fetch('https://mon-site.com/oauth/token', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: new URLSearchParams({
    grant_type: 'client_credentials',
    client_id: process.env.DRUPAL_CLIENT_ID,
    client_secret: process.env.DRUPAL_CLIENT_SECRET,
  }),
});

const { access_token, expires_in } = await tokenResponse.json();

// Utiliser le token pour appeler JSON:API
const response = await fetch('https://mon-site.com/jsonapi/node/article', {
  headers: {
    'Authorization': `Bearer ${access_token}`,
    'Accept': 'application/vnd.api+json',
  },
});
```

---

## Authorization Code + PKCE — Applications Frontend

Pour les applications où un utilisateur se connecte (Next.js, SPA React...) :

```javascript
// 1. Générer le code_verifier et code_challenge (PKCE)
function generateCodeVerifier() {
  const array = new Uint8Array(32);
  crypto.getRandomValues(array);
  return btoa(String.fromCharCode(...array))
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=/g, '');
}

async function generateCodeChallenge(verifier) {
  const encoder = new TextEncoder();
  const data = encoder.encode(verifier);
  const digest = await crypto.subtle.digest('SHA-256', data);
  return btoa(String.fromCharCode(...new Uint8Array(digest)))
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=/g, '');
}

// 2. Rediriger vers Drupal pour l'authentification
const codeVerifier = generateCodeVerifier();
const codeChallenge = await generateCodeChallenge(codeVerifier);

// Stocker le verifier en session
sessionStorage.setItem('code_verifier', codeVerifier);

const authUrl = new URL('https://mon-site.com/oauth/authorize');
authUrl.searchParams.set('response_type', 'code');
authUrl.searchParams.set('client_id', CLIENT_ID);
authUrl.searchParams.set('redirect_uri', 'https://mon-frontend.com/callback');
authUrl.searchParams.set('scope', 'authenticated');
authUrl.searchParams.set('code_challenge', codeChallenge);
authUrl.searchParams.set('code_challenge_method', 'S256');

window.location.href = authUrl.toString();

// 3. Dans la page callback — échanger le code contre un token
const urlParams = new URLSearchParams(window.location.search);
const code = urlParams.get('code');
const codeVerifier = sessionStorage.getItem('code_verifier');

const tokenResponse = await fetch('https://mon-site.com/oauth/token', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: new URLSearchParams({
    grant_type: 'authorization_code',
    client_id: CLIENT_ID,
    redirect_uri: 'https://mon-frontend.com/callback',
    code: code,
    code_verifier: codeVerifier,
  }),
});

const { access_token, refresh_token, expires_in } = await tokenResponse.json();
```

---

## Refresh Token — Renouveler sans Reconnexion

```javascript
// Renouveler l'access token avec le refresh token
async function refreshAccessToken(refreshToken) {
  const response = await fetch('https://mon-site.com/oauth/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'refresh_token',
      refresh_token: refreshToken,
      client_id: CLIENT_ID,
      // client_secret requis uniquement pour les clients confidentiels
    }),
  });

  if (!response.ok) {
    // Refresh token expiré → rediriger vers la connexion
    window.location.href = '/login';
    return null;
  }

  return await response.json();
}
```

---

## Configuration CORS pour le Frontend

```yaml
# web/sites/default/services.yml
parameters:
  cors.config:
    enabled: true
    allowedHeaders:
      - 'x-csrf-token'
      - 'authorization'
      - 'content-type'
      - 'accept'
      - 'origin'
      - 'x-requested-with'
    allowedMethods:
      - 'GET'
      - 'POST'
      - 'PATCH'
      - 'DELETE'
      - 'OPTIONS'
    allowedOrigins:
      # ❌ NE PAS faire :
      # - '*'    → ouvert à tous les domaines
      # ✅ Whitelist précise :
      - 'https://mon-frontend.com'
      - 'https://staging.mon-frontend.com'
      - 'http://localhost:3000'    # Dev local uniquement
    exposedHeaders: false
    maxAge: false
    supportsCredentials: true    # Requis pour les cookies de session
```

---

## Sécuriser les Routes API

```php
// hook_jsonapi_entity_filter_access() — accès par token OAuth
function mon_module_jsonapi_entity_filter_access(
  \Drupal\Core\Entity\EntityTypeInterface $entity_type,
  \Drupal\Core\Session\AccountInterface $account
): array {
  // Permettre les filtres uniquement aux utilisateurs authentifiés
  // (rôle attribué via Simple OAuth)
  if ($account->hasRole('api_consumer_role')) {
    return [
      \Drupal\jsonapi\Access\EntityAccessChecker::FILTER_ON_DATA_FIELD_ACCESS
        => \Drupal\Core\Access\AccessResult::allowed(),
    ];
  }

  return [];
}

// Vérifier le token OAuth dans un Controller custom
use Drupal\simple_oauth\Authentication\TokenAuthUser;

class MonApiController extends ControllerBase {
  public function maRoute(): Response {
    $account = \Drupal::currentUser();

    // Vérifier que l'accès est via OAuth (pas session browser)
    if (!($account instanceof TokenAuthUser)) {
      throw new \Symfony\Component\HttpKernel\Exception\UnauthorizedHttpException(
        'Bearer',
        'Token OAuth requis.'
      );
    }

    // Vérifier les scopes du token
    $scopes = $account->getToken()->get('scopes')->getValue();
    // ...
  }
}
```
