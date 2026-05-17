---
name: drupal-api — JSON:API référence complète
description: Référence complète de JSON:API Drupal - filtres (simples, groupes AND/OR, opérateurs), includes, sparse fieldsets, pagination, tri, mutations POST/PATCH/DELETE, sécurisation, et cache.
---

# JSON:API Drupal — Référence Complète

## Endpoints JSON:API

```
GET  /jsonapi/{entity_type}/{bundle}           # Liste d'entités
GET  /jsonapi/{entity_type}/{bundle}/{uuid}    # Entité spécifique
POST /jsonapi/{entity_type}/{bundle}           # Créer une entité
PATCH /jsonapi/{entity_type}/{bundle}/{uuid}   # Modifier une entité
DELETE /jsonapi/{entity_type}/{bundle}/{uuid}  # Supprimer une entité

# Exemples :
GET  /jsonapi/node/article                     # Tous les articles
GET  /jsonapi/node/article/{uuid}              # Un article
GET  /jsonapi/taxonomy_term/tags               # Termes de taxonomy
GET  /jsonapi/user/user                        # Utilisateurs
GET  /jsonapi/media/image                      # Médias images
GET  /jsonapi/paragraph/section                # Paragraphes section
```

---

## Filtres — Référence Complète

### Filtres Simples

```bash
# Filtre d'égalité simple
GET /jsonapi/node/article?filter[status]=1

# Filtre sur un champ imbriqué (via relationship)
GET /jsonapi/node/article?filter[field_tags.name]=Drupal

# Filtre sur la langue
GET /jsonapi/node/article?filter[langcode]=fr

# Filtre sur le type de bundle
GET /jsonapi/node/article?filter[type]=article
```

### Filtres avec Opérateurs

```bash
# Format : filter[ALIAS][condition][path]
#          filter[ALIAS][condition][value]
#          filter[ALIAS][condition][operator]

# Supérieur à
GET /jsonapi/node/article?filter[big][condition][path]=field_prix&filter[big][condition][value]=100&filter[big][condition][operator]=GREATER_THAN

# CONTAINS (LIKE %value%)
GET /jsonapi/node/article?filter[search][condition][path]=title&filter[search][condition][value]=Drupal&filter[search][condition][operator]=CONTAINS

# STARTS_WITH
GET /jsonapi/node/article?filter[start][condition][path]=title&filter[start][condition][value]=Drupal&filter[start][condition][operator]=STARTS_WITH

# IN (liste de valeurs)
GET /jsonapi/node/article?filter[ids][condition][path]=nid&filter[ids][condition][value][]=1&filter[ids][condition][value][]=2&filter[ids][condition][value][]=3&filter[ids][condition][operator]=IN

# IS NULL
GET /jsonapi/node/article?filter[no-image][condition][path]=field_image.id&filter[no-image][condition][operator]=IS%20NULL

# Opérateurs disponibles :
# = | != | < | > | <= | >= | CONTAINS | STARTS_WITH | ENDS_WITH | IN | NOT IN | BETWEEN | IS NULL | IS NOT NULL
```

### Groupes de Filtres — AND / OR

```bash
# AND implicite (tous les filtres à la racine sont ET)
GET /jsonapi/node/article?filter[status]=1&filter[langcode]=fr

# OR explicite — articles en FR ou EN
GET /jsonapi/node/article?filter[lang-group][group][conjunction]=OR&filter[fr][condition][path]=langcode&filter[fr][condition][value]=fr&filter[fr][condition][memberOf]=lang-group&filter[en][condition][path]=langcode&filter[en][condition][value]=en&filter[en][condition][memberOf]=lang-group

# Combinaison complexe : (status=1 AND type=article) OR (status=1 AND type=page)
# Utiliser des groupes imbriqués avec memberOf
```

---

## Includes — Eager Loading de Relations

```bash
# Inclure les termes de taxonomy liés
GET /jsonapi/node/article?include=field_tags

# Inclure plusieurs relations
GET /jsonapi/node/article?include=field_tags,field_author,field_image

# Relations imbriquées (auteur + rôles de l'auteur)
GET /jsonapi/node/article?include=field_author.roles

# Image + fichier de l'image
GET /jsonapi/node/article?include=field_image.field_media_image

# Paragraphs inclus dans le nœud
GET /jsonapi/node/article?include=field_contenu,field_contenu.field_image
```

**Réponse** : les entités incluses apparaissent dans `data.relationships` et `included`.

---

## Sparse Fieldsets — Limiter les Champs Retournés

```bash
# Retourner uniquement title et body (réduit la taille de la réponse)
GET /jsonapi/node/article?fields[node--article]=title,body,field_image

# Champs sur une relation incluse aussi
GET /jsonapi/node/article?include=field_tags&fields[node--article]=title,field_tags&fields[taxonomy_term--tags]=name
```

---

## Pagination

```bash
# Offset-based (défaut Drupal)
GET /jsonapi/node/article?page[offset]=0&page[limit]=10    # Page 1
GET /jsonapi/node/article?page[offset]=10&page[limit]=10   # Page 2

# Cursor-based (via jsonapi_extras ou contrib)
# Non natif Drupal core

# La réponse inclut des liens de pagination :
# links.next, links.prev, links.first, links.last

# Limite maximum configurable via settings
# (/admin/config/services/jsonapi) ou code
```

---

## Tri

```bash
# Tri ascendant par titre
GET /jsonapi/node/article?sort=title

# Tri descendant (préfixe -)
GET /jsonapi/node/article?sort=-created

# Tri multiple
GET /jsonapi/node/article?sort=-created,title

# Tri sur un champ de relation
GET /jsonapi/node/article?sort=field_author.name
```

---

## Mutations — POST / PATCH / DELETE

### En-têtes Requis

```http
Content-Type: application/vnd.api+json
Accept: application/vnd.api+json
X-CSRF-Token: {token}   ← obtenu via /session/token
```

Obtenir le token CSRF :
```bash
curl https://mon-site.com/session/token
# Réponse : 1234abcd5678efgh (le token)
```

### POST — Créer un Nœud

```json
POST /jsonapi/node/article
Content-Type: application/vnd.api+json

{
  "data": {
    "type": "node--article",
    "attributes": {
      "title": "Mon nouvel article",
      "body": {
        "value": "<p>Contenu de l'article.</p>",
        "format": "basic_html"
      },
      "status": true,
      "langcode": "fr"
    },
    "relationships": {
      "field_tags": {
        "data": [
          {
            "type": "taxonomy_term--tags",
            "id": "uuid-du-terme"
          }
        ]
      }
    }
  }
}
```

### PATCH — Modifier un Nœud

```json
PATCH /jsonapi/node/article/{uuid}
Content-Type: application/vnd.api+json

{
  "data": {
    "type": "node--article",
    "id": "{uuid}",
    "attributes": {
      "title": "Titre modifié"
    }
  }
}
```

**Note :** seuls les champs présents dans le payload sont modifiés (PATCH partiel).

### DELETE — Supprimer

```http
DELETE /jsonapi/node/article/{uuid}
```

Réponse : `204 No Content` si succès.

---

## Sécurisation des Endpoints

### Permissions par Bundle

```php
// hook_jsonapi_entity_filter_access() — sécuriser les filtres
function mon_module_jsonapi_entity_filter_access(\Drupal\Core\Entity\EntityTypeInterface $entity_type, \Drupal\Core\Session\AccountInterface $account): array {
  if ($entity_type->id() === 'node') {
    return [
      \Drupal\jsonapi\Access\EntityAccessChecker::FILTER_ON_DATA_FIELD_ACCESS => AccessResult::allowedIfHasPermission($account, 'access content'),
    ];
  }
  return [];
}

// hook_jsonapi_ENTITY_TYPE_filter_access() — plus spécifique
function mon_module_jsonapi_node_filter_access(\Drupal\node\NodeInterface $node, \Drupal\Core\Session\AccountInterface $account, string $operation): \Drupal\Core\Access\AccessResultInterface {
  // Logique d'accès custom
  return AccessResult::allowedIfHasPermission($account, 'access content');
}
```

### Masquer des Champs (jsonapi_extras)

```yaml
# config/install/jsonapi_resource_config.node--article.yml
langcode: fr
status: true
id: node--article
resourceType: node--article
resourceFields:
  field_internal_notes:
    fieldName: field_internal_notes
    publicName: field_internal_notes
    enhancer:
      id: ''
    disabled: true    # ← champ masqué dans l'API
```

---

## Cache JSON:API

JSON:API bénéficie automatiquement du système de cache tags Drupal. Les réponses sont mises en cache avec les tags des entités retournées.

```bash
# Headers de cache dans la réponse
X-Drupal-Cache-Tags: node:42 node_list user:1
X-Drupal-Cache-Contexts: languages:language_interface url.query_args user.roles
X-Drupal-Dynamic-Cache: MISS  # MISS = pas encore en cache, HIT = depuis le cache

# Invalider depuis Varnish quand ces tags changent
# → Varnish lit X-Drupal-Cache-Tags et invalide lors des purges
```

---

## Exemples JavaScript (Fetch API)

```javascript
// GET — lire des articles publiés en français
const response = await fetch(
  '/jsonapi/node/article?' + new URLSearchParams({
    'filter[status]': '1',
    'filter[langcode]': 'fr',
    'sort': '-created',
    'page[limit]': '10',
    'include': 'field_tags',
    'fields[node--article]': 'title,body,field_tags,created',
  }),
  {
    headers: {
      'Accept': 'application/vnd.api+json',
    },
  }
);
const data = await response.json();

// POST — créer un nœud (authentifié)
const token = await fetch('/session/token').then(r => r.text());

const created = await fetch('/jsonapi/node/article', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/vnd.api+json',
    'Accept': 'application/vnd.api+json',
    'X-CSRF-Token': token,
  },
  body: JSON.stringify({
    data: {
      type: 'node--article',
      attributes: {
        title: 'Mon article via API',
        status: true,
      },
    },
  }),
});
```
