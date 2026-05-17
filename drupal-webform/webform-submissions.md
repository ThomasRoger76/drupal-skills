---
name: drupal-webform — soumissions
description: Accéder et manipuler les soumissions Webform depuis PHP, exporter les résultats, hooks de soumission, et gestion de la rétention des données.
---

# Webform Submissions — API PHP & Gestion

## Lire les Soumissions

```php
use Drupal\webform\Entity\WebformSubmission;

// Charger une soumission par ID
$submission = WebformSubmission::load($sid);

// Obtenir les données de la soumission
$data = $submission->getData();
$prenom = $data['prenom'] ?? '';
$email = $data['email'] ?? '';

// Métadonnées de la soumission
$webform_id = $submission->getWebform()->id();
$created = $submission->getCreatedTime();
$uid = $submission->getOwnerId();
$ip = $submission->getRemoteAddr();
$state = $submission->getState();  // 'completed', 'draft', 'locked'

// Charger toutes les soumissions d'un formulaire
$sids = \Drupal::entityQuery('webform_submission')
  ->condition('webform_id', 'mon_formulaire')
  ->condition('in_draft', FALSE)
  ->sort('created', 'DESC')
  ->range(0, 50)
  ->accessCheck(FALSE)
  ->execute();

$submissions = WebformSubmission::loadMultiple($sids);

// Lister les soumissions d'un utilisateur
$sids = \Drupal::entityQuery('webform_submission')
  ->condition('webform_id', 'mon_formulaire')
  ->condition('uid', $uid)
  ->accessCheck(TRUE)
  ->execute();
```

---

## Créer une Soumission Programmatiquement

```php
use Drupal\webform\Entity\Webform;
use Drupal\webform\Entity\WebformSubmission;

// Créer et sauvegarder une soumission
$webform = Webform::load('mon_formulaire');

$submission = WebformSubmission::create([
  'webform_id' => 'mon_formulaire',
  'webform' => $webform,
  'uid' => \Drupal::currentUser()->id(),
  'data' => [
    'prenom' => 'Jean',
    'nom' => 'Dupont',
    'email' => 'jean@example.com',
    'message' => 'Demande créée programmatiquement',
  ],
]);

// Déclencher les handlers (email, remote post...)
$submission->save();

echo 'Soumission créée : SID ' . $submission->id();

// Créer SANS déclencher les handlers
$submission->setData(['prenom' => 'Jean', 'email' => 'jean@example.com']);
$submission->resave();  // resave = pas de re-trigger des handlers
```

---

## Modifier une Soumission Existante

```php
$submission = WebformSubmission::load($sid);

// Modifier une valeur
$data = $submission->getData();
$data['statut'] = 'traite';
$submission->setData($data);

// Modifier un seul élément
$submission->setElementData('statut', 'traite');

// Changer l'état
$submission->setState('locked');  // verrouiller la soumission

// Sauvegarder sans re-déclencher les handlers
$submission->resave();
```

---

## Hooks Webform

```php
<?php

/**
 * Implements hook_webform_submission_presave().
 *
 * Appelé avant chaque sauvegarde de soumission.
 */
function mon_module_webform_submission_presave(WebformSubmissionInterface $webform_submission): void {
  if ($webform_submission->getWebform()->id() !== 'mon_formulaire') {
    return;
  }

  // Calculer un champ calculé côté serveur
  $data = $webform_submission->getData();
  $data['nom_complet'] = trim(($data['prenom'] ?? '') . ' ' . ($data['nom'] ?? ''));
  $webform_submission->setData($data);
}

/**
 * Implements hook_webform_submission_insert().
 *
 * Appelé après chaque nouvelle soumission.
 */
function mon_module_webform_submission_insert(WebformSubmissionInterface $webform_submission): void {
  if ($webform_submission->getWebform()->id() !== 'devis_request') {
    return;
  }

  // Créer un nœud "Devis" depuis la soumission
  $data = $webform_submission->getData();

  $node = \Drupal\node\Entity\Node::create([
    'type' => 'devis',
    'title' => 'Devis pour ' . $data['nom_entreprise'],
    'field_contact_email' => $data['email'],
    'field_submission_id' => $webform_submission->id(),
    'status' => 0,  // Brouillon
  ]);
  $node->save();
}

/**
 * Implements hook_webform_submission_form_alter().
 *
 * Modifier le formulaire rendu.
 */
function mon_module_webform_submission_form_alter(
  array &$form,
  FormStateInterface $form_state,
  string $form_id
): void {
  // Uniquement pour les formulaires Webform
  if (!str_starts_with($form_id, 'webform_submission_')) {
    return;
  }

  $webform = $form_state->getFormObject()->getWebform();
  if ($webform->id() !== 'mon_formulaire') {
    return;
  }

  // Ajouter une classe CSS custom
  $form['#attributes']['class'][] = 'mon-formulaire-custom';

  // Modifier un label
  if (isset($form['elements']['email'])) {
    $form['elements']['email']['#title'] = t('Votre adresse email professionnelle');
  }
}
```

---

## Rétention des Données (RGPD)

```
Webform → Settings → Submissions

Configuration :
  ├── Store submissions : ✅
  ├── Limit total submissions : ex: 1000 max
  ├── Purge : Completed | Draft | All
  └── Purge after : 90 jours (pour la conformité RGPD)
```

```bash
# Purger manuellement les soumissions anciennes
drush php:eval "
\$webform = \Drupal\webform\Entity\Webform::load('contact');
\$storage = \Drupal::entityTypeManager()->getStorage('webform_submission');

\$cutoff = strtotime('-90 days');
\$sids = \$storage->getQuery()
  ->condition('webform_id', 'contact')
  ->condition('created', \$cutoff, '<')
  ->condition('in_draft', FALSE)
  ->accessCheck(FALSE)
  ->execute();

if (\$sids) {
  \$submissions = \$storage->loadMultiple(\$sids);
  \$storage->delete(\$submissions);
  echo count(\$sids) . ' soumission(s) supprimée(s).';
}
"
```

---

## Export CSV

```bash
# Via l'UI : /admin/structure/webform/{id}/results/download
# Format : CSV, Excel, JSON
# Filtres : période, état (completed/draft/all)

# Via Drush
drush php:eval "
\$exporter = \Drupal::service('webform.results_exporter');
\$webform = \Drupal\webform\Entity\Webform::load('contact');
\$export = \$exporter->getSubmissionsColumns(\$webform);
echo count(\$export) . ' colonnes disponibles.';
"
```
