---
name: drupal-token — tokens dans les modules
description: Utiliser les tokens dans Metatag, Pathauto, Webform, et les templates d'email Drupal - patterns recommandés et tokens complexes.
---

# Tokens dans les Modules — Référence Pratique

## Tokens dans Metatag

```
/admin/config/search/metatag → Content: Article

Tokens essentiels :
  title:       [node:title] | [site:name]
  description: [node:summary|node:body:summary]
  og:title:    [node:title]
  og:image:    [node:field_image:entity:field_media_image:entity:url]  ← Media
               [node:field_image:entity:file:url]                      ← File direct
  og:image:alt:[node:field_image:entity:field_media_image:alt]
  canonical:   [node:url]
  
  # Schema.org
  datePublished: [node:created:html_datetime]
  dateModified:  [node:changed:html_datetime]
  author:        [node:author:name]
```

---

## Tokens dans Pathauto

```
/admin/config/search/path/patterns

Patterns par type de contenu :

# Article simple
articles/[node:title]

# Article avec catégorie
articles/[node:field_categorie:entity:name]/[node:title]

# Événement avec date
evenements/[node:field_date:value:custom:Y/m]/[node:title]

# Produit avec référence
produits/[node:field_reference]/[node:title]

# Terme de taxonomie avec hiérarchie
[term:parents:join-path]/[term:name]

# Utilisateur
utilisateurs/[user:name]
```

---

## Tokens dans Webform (Handlers Email)

```
Handler Email → Subject et Body

Tokens disponibles automatiquement dans Webform :
  [webform:title]                         → Nom du formulaire
  [webform_submission:sid]                → ID de la soumission
  [webform_submission:created]            → Date de soumission
  [webform_submission:remote_addr]        → IP du soumettant
  
  # Valeurs des champs
  [webform_submission:values:prenom]      → Valeur du champ "prenom"
  [webform_submission:values:email]       → Valeur du champ "email"
  [webform_submission:values]             → Toutes les valeurs en table HTML
  
  # Utilisateur connecté (si authentifié)
  [current-user:name]
  [current-user:mail]
  
  # Site
  [site:name]
  [site:url]

# Exemples :
Subject: "Nouvelle demande de [webform_submission:values:prenom] [webform_submission:values:nom]"
Body:    "Bonjour, [webform_submission:values:prenom],\nVotre message a été reçu..."
```

---

## Tokens dans les Templates Email (Symfony Mailer)

```php
// Dans l'EmailBuilder — passer les tokens via variables
public function build(EmailInterface $email): void {
  $commande = $email->getParam('commande');

  // Remplacement direct des tokens dans le sujet
  $token_service = \Drupal::service('token');
  $subject = $token_service->replace(
    'Commande [commande:reference] confirmée',
    ['commande' => $commande],
    ['clear' => TRUE]
  );
  $email->setSubject($subject);

  // Passer l'entité au template pour utilisation dans Twig
  $email->setVariable('commande', $commande);
}
```

```twig
{# Dans le template email — utiliser les variables Twig directement #}
{# (pas besoin de tokens dans les templates Twig — Twig est plus expressif) #}

Commande : {{ commande.reference.value }}
Total : {{ commande.montant.value|number_format(2, ',', ' ') }} €
Client : {{ commande.get('uid').entity.displayname }}
```

---

## Tokens Complexes — Exemples Réels

```
# Accéder au premier tag d'un article (index 0)
[node:field_tags:0:entity:name]

# Jointure de plusieurs valeurs (séparées par virgule)
[node:field_tags:join:, ]

# URL d'une image avec style
[node:field_image:entity:field_media_image:entity:url]

# Alt text d'une image Media
[node:field_image:entity:field_media_image:alt]

# Token d'un sous-champ (date formatée)
[node:field_date:value:custom:d/m/Y]
[node:field_date:value:custom:Y-m-d\TH:i:sP]  ← ISO 8601 pour Schema.org

# Token d'entité référencée → champ → valeur
[node:field_auteur:entity:field_poste:value]

# Token avec fallback (chain)
[node:field_resume|node:body:summary|Pas de résumé.]
```
