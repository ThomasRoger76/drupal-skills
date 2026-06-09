---
name: drupal-webform — API REST
description: Utiliser l'API REST Webform pour soumettre des formulaires depuis un frontend headless, récupérer les définitions, et intégrer Webform dans des applications découplées.
---

# Webform API REST — Référence Complète

## Activation de l'API REST Webform

```bash
# Modules requis (hal est déprécié/retiré en D10+ : ne l'installer que si une
# intégration legacy l'exige ; rest + serialization suffisent pour du JSON)
docker compose exec php drush en webform_rest serialization basic_auth -y

# Vérifier que l'endpoint est actif
curl https://mon-site.com/webform_rest/mon_formulaire/fields?_format=json
```

---

## Endpoints Disponibles

```bash
# GET — Récupérer la définition d'un formulaire (champs, options)
GET /webform_rest/{webform_id}/fields?_format=json

# POST — Soumettre un formulaire
POST /webform_rest/submit
Content-Type: application/json

# GET — Récupérer une soumission (avec token ou auth)
GET /webform_rest/{webform_id}/submission/{sid}?_format=json
```

---

## Récupérer la Définition du Formulaire

```bash
# Obtenir les champs et leurs types
curl https://mon-site.com/webform_rest/contact/fields?_format=json

# Réponse exemple :
{
  "prenom": {
    "#type": "textfield",
    "#title": "Prénom",
    "#required": true,
    "#maxlength": 100
  },
  "email": {
    "#type": "email",
    "#title": "Email",
    "#required": true
  },
  "sujet": {
    "#type": "select",
    "#title": "Sujet",
    "#options": {
      "info": "Demande d'information",
      "support": "Support technique"
    }
  }
}
```

---

## Soumettre un Formulaire via REST

```bash
# Sans authentification (si le formulaire est public)
curl -X POST https://mon-site.com/webform_rest/submit \
  -H "Content-Type: application/json" \
  -d '{
    "webform_id": "contact",
    "prenom": "Jean",
    "nom": "Dupont",
    "email": "jean@example.com",
    "sujet": "info",
    "message": "Bonjour, je voudrais en savoir plus."
  }'

# Avec authentification Basic (si formulaire réservé aux connectés)
curl -X POST https://mon-site.com/webform_rest/submit \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic $(echo -n 'user:password' | base64)" \
  -d '{"webform_id": "contact", ...}'

# Réponse succès :
{"sid": "42", "confirmation_message": "Merci pour votre message."}

# Réponse erreur validation :
{"error": {"email": "L'adresse email n'est pas valide."}}
```

---

## Intégration JavaScript (Next.js / Vue.js)

```typescript
// Soumettre un formulaire Webform depuis un frontend
async function submitWebform(formId: string, data: Record<string, any>) {
  const payload = {
    webform_id: formId,
    ...data,
  };

  const response = await fetch(`${process.env.DRUPAL_BASE_URL}/webform_rest/submit`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(payload),
  });

  const result = await response.json();

  if (response.ok && result.sid) {
    return { success: true, submissionId: result.sid, message: result.confirmation_message };
  }

  if (result.error) {
    return { success: false, errors: result.error };
  }

  throw new Error('Erreur inconnue lors de la soumission');
}

// Utilisation dans un composant React/Next.js
const handleSubmit = async (formData: FormData) => {
  const result = await submitWebform('contact', {
    prenom: formData.get('prenom'),
    email: formData.get('email'),
    message: formData.get('message'),
  });

  if (result.success) {
    setConfirmationMessage(result.message);
  } else {
    setErrors(result.errors);
  }
};
```

---

## Récupérer les Soumissions via REST

```bash
# Récupérer une soumission avec son token (email de confirmation)
GET /webform_rest/contact/submission/42?token=abc123&_format=json

# Avec authentification (admin/éditeur)
curl https://mon-site.com/webform_rest/contact/submission/42?_format=json \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)"

# Réponse :
{
  "sid": "42",
  "webform_id": "contact",
  "created": "2026-05-16T10:30:00+00:00",
  "uid": "15",
  "data": {
    "prenom": "Jean",
    "email": "jean@example.com",
    "message": "Bonjour..."
  }
}
```

---

## Configuration des Permissions REST

```
/admin/config/services/rest

→ Enable "Webform Submit" resource
→ Methods : POST
→ Formats : json
→ Authentication : basic_auth, cookie (selon le besoin)
```

```yaml
# config/install/rest.resource.webform_rest_submit.yml
langcode: fr
status: true
id: webform_rest_submit
plugin_id: webform_rest_submit
granularity: resource
configuration:
  methods:
    - POST
  formats:
    - json
  authentication:
    - cookie
    - basic_auth
```

---

## Sécuriser l'API Webform

```php
// Ajouter une validation du token CSRF pour les soumissions REST
// Automatiquement géré si cookie authentication est utilisé

// Pour basic_auth — pas de CSRF token nécessaire
// Pour cookie auth (frontend Drupal) — X-CSRF-Token requis

// Obtenir le token CSRF
curl https://mon-site.com/session/token
# → "TOKEN_STRING"

// Utiliser dans la soumission
curl -X POST https://mon-site.com/webform_rest/submit \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: TOKEN_STRING" \
  -H "Cookie: DRUPAL_SESSION_COOKIE" \
  -d '{"webform_id": "contact", ...}'
```

---

## Webform comme Source de Données (Headless)

```typescript
// Charger la structure d'un formulaire pour le rendre côté frontend
async function loadWebformStructure(formId: string) {
  const response = await fetch(
    `${DRUPAL_URL}/webform_rest/${formId}/fields?_format=json`
  );
  const fields = await response.json();

  // Transformer la structure Drupal en composants React
  return Object.entries(fields)
    .filter(([key]) => !key.startsWith('#'))
    .map(([key, field]: [string, any]) => ({
      id: key,
      type: field['#type'],
      label: field['#title'],
      required: field['#required'] ?? false,
      options: field['#options'] ?? null,
      placeholder: field['#placeholder'] ?? null,
    }));
}
```
