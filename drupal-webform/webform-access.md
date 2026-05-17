---
name: drupal-webform — access control
description: Contrôle d'accès aux formulaires Webform Drupal - permissions par rôle, limites de soumissions, accès par token, et accès programmatique.
---

# Webform Access Control — Référence Complète

## Configuration des Permissions

```
Webform → Edit → Access (onglet)

Sections d'accès configurables :
  ├── Create submissions (soumettre le formulaire)
  │     → Qui peut remplir et soumettre ce formulaire ?
  ├── View own submissions (voir ses soumissions)
  ├── View any submissions (voir toutes les soumissions)
  ├── Update own submissions (modifier ses soumissions)
  ├── Update any submissions
  ├── Delete own submissions
  └── Delete any submissions

Pour chaque section :
  ├── Anonymous users : ☐ (non recommandé pour les formulaires sensibles)
  ├── Authenticated users : ✅ (utilisateurs connectés)
  ├── Specific roles : editor, administrator
  └── Specific users : UIDs spécifiques
```

---

## Limiter le Nombre de Soumissions

```
Webform → Settings → Submissions

Limites disponibles :
  ├── Limit total submissions : 500 (ex: inscriptions limitées)
  ├── Limit submissions per user : 1 (ex: une seule candidature par personne)
  ├── Message si limite atteinte : "Les inscriptions sont fermées."
  └── Limit by interval : 10 par jour, 50 par semaine
```

---

## Accès par Lien Token (Formulaires Privés)

```
Webform → Settings → Access → "Open" (pas de restriction)
+ Génération d'un lien avec token unique par utilisateur

Utilisation :
  → Envoyer un lien unique dans un email
  → L'utilisateur accède directement à "sa" soumission
  → Impossible de deviner les soumissions des autres
```

```php
// Générer un lien tokenisé vers une soumission
$webform_submission = \Drupal\webform\Entity\WebformSubmission::load($sid);
$token = \Drupal::token()->generate(
  'webform_submission',
  ['webform_submission' => $webform_submission]
);

$url = \Drupal\Core\Url::fromRoute(
  'entity.webform_submission.user',
  [
    'webform' => $webform_submission->getWebform()->id(),
    'webform_submission' => $sid,
  ],
  ['query' => ['token' => $token], 'absolute' => TRUE]
);
```

---

## hook_webform_access() — Contrôle Programmatique

```php
<?php

/**
 * Implements hook_webform_access().
 *
 * Contrôler l'accès à un formulaire Webform depuis le code.
 */
function mon_module_webform_access(
  \Drupal\webform\WebformInterface $webform,
  string $operation,
  \Drupal\Core\Session\AccountInterface $account
): \Drupal\Core\Access\AccessResultInterface {

  // Uniquement pour notre formulaire
  if ($webform->id() !== 'inscription_evenement') {
    return \Drupal\Core\Access\AccessResult::neutral();
  }

  // Vérifier si l'événement est encore ouvert
  $config = \Drupal::config('mon_module.evenement_settings');
  $date_limite = $config->get('inscription_date_limite');

  if ($date_limite && time() > $date_limite) {
    if ($operation === 'create') {
      return \Drupal\Core\Access\AccessResult::forbidden()
        ->addCacheTags(['config:mon_module.evenement_settings']);
    }
  }

  // Vérifier si l'utilisateur a déjà soumis (limite 1 par utilisateur)
  if ($operation === 'create' && $account->isAuthenticated()) {
    $existing = \Drupal::entityQuery('webform_submission')
      ->condition('webform_id', 'inscription_evenement')
      ->condition('uid', $account->id())
      ->accessCheck(FALSE)
      ->count()
      ->execute();

    if ($existing > 0) {
      return \Drupal\Core\Access\AccessResult::forbidden()
        ->cachePerUser();
    }
  }

  return \Drupal\Core\Access\AccessResult::neutral();
}
```

---

## Affichage Conditionnel du Formulaire

```twig
{# Dans un template — afficher le formulaire seulement si l'utilisateur peut soumettre #}
{% if webform.isOpen and webform.access('create') %}
  {{ drupal_entity('webform', webform_id) }}
{% elseif webform.isClosed %}
  <p class="webform-closed">{{ 'Les inscriptions sont fermées.'|t }}</p>
{% else %}
  <p>{{ 'Vous devez être connecté pour accéder à ce formulaire.'|t }}</p>
  <a href="{{ path('user.login', {}, {'query': {'destination': url('<current>')}}) }}">
    {{ 'Se connecter'|t }}
  </a>
{% endif %}
```

---

## Permissions Globales Webform

```
/admin/people/permissions → section "Webform"

Permissions importantes :
  ├── "Administer webform" → tout gérer
  ├── "Create any webform" → créer des formulaires
  ├── "Edit own webform submissions" → modifier ses soumissions
  ├── "View own webform submissions" → voir ses soumissions
  ├── "Access webform overview" → voir la liste des formulaires (admin)
  └── "Access webform results" → voir les soumissions (admin)
```

---

## Fermer/Ouvrir un Formulaire Programmatiquement

```php
use Drupal\webform\Entity\Webform;

// Fermer un formulaire (après la date limite)
$webform = Webform::load('inscription_evenement');
$webform->setStatus(FALSE);   // FALSE = fermé
$webform->save();

// Réouvrir
$webform->setStatus(TRUE);
$webform->save();

// Fermer via Drush
drush php:eval "
\$webform = \Drupal\webform\Entity\Webform::load('inscription_evenement');
\$webform->setStatus(FALSE)->save();
echo 'Formulaire fermé.';
"

// Vérifier l'état
drush php:eval "
\$webform = \Drupal\webform\Entity\Webform::load('inscription_evenement');
echo \$webform->isOpen() ? 'Ouvert' : 'Fermé';
"
```
