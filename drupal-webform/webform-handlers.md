---
name: drupal-webform — handlers
description: Configurer les handlers Webform (Email, Remote POST, Action) et créer des handlers custom avec WebformHandlerBase.
---

# Webform Handlers — Référence Complète

## Types de Handlers

```
/admin/structure/webform/{id}/handlers/add

Handlers disponibles :
├── Email           → Envoyer un email à l'utilisateur ou l'admin
├── Remote post     → Envoyer les données à une URL externe (webhook, CRM)
├── Action          → Déclencher une Action Drupal
├── Save as Draft   → Permettre à l'utilisateur de sauvegarder son brouillon
├── Debug           → Logger les soumissions (dev uniquement)
└── Custom          → Plugin WebformHandlerBase
```

---

## Handler Email

```
Settings :
  ├── To : [webform_submission:values:email]  ← email de l'utilisateur
  ├── CC / BCC : admin@mon-site.com
  ├── Reply-to : [webform_submission:values:email]
  ├── Subject : "Confirmation de votre demande - [webform:title]"
  ├── Body : utilise un template Twig ou l'auto-generated message
  ├── HTML format : ✅ recommandé
  ├── Conditions : (toujours / si champ X = Y)
  └── States : completed / draft / ...
```

```yaml
# Configuration YAML d'un handler Email
handlers:
  email_confirmation:
    id: email
    label: 'Email de confirmation'
    notes: ''
    handler_id: email
    status: true
    conditions: []
    weight: 0
    settings:
      states:
        - completed
      to_mail: '[webform_submission:values:email]'
      to_options: {  }
      bcc_mail: ''
      cc_mail: ''
      from_mail: '[site:mail]'
      from_name: '[site:name]'
      reply_to: ''
      return_path: ''
      sender_mail: ''
      sender_name: ''
      subject: 'Confirmation de votre demande — [webform:title]'
      body: |
        Bonjour [webform_submission:values:prenom],

        Nous avons bien reçu votre demande.

        Récapitulatif :
        [webform_submission:values]

        Cordialement,
        L'équipe [site:name]
      html: true
      attachments: false
      excluded_elements: { }
      ignore_access: false
      exclude_empty: true
      exclude_empty_checkbox: false
      debug: false
```

---

## Handler Remote POST (Webhook / CRM)

```yaml
handlers:
  crm_integration:
    id: remote_post
    label: 'Intégration CRM'
    handler_id: remote_post
    status: true
    conditions: []
    weight: 1
    settings:
      states:
        - completed
      completed_url: 'https://crm.example.com/api/leads'
      completed_custom_data: |
        {
          "first_name": "[webform_submission:values:prenom]",
          "last_name": "[webform_submission:values:nom]",
          "email": "[webform_submission:values:email]",
          "source": "drupal_webform",
          "form": "[webform:title]"
        }
      completed_type: json          # json ou x-www-form-urlencoded
      method: POST
      file_data: false
      message: ''
      messages: { }
      excluded_elements: { }
      ignore_access: false
      exclude_empty: true
      # Authentification Bearer
      completed_custom_headers: |
        Authorization: Bearer MON_TOKEN_API
```

---

## Handler Custom — WebformHandlerBase

> **Drupal 11 / Webform 6.2+ : attribut PHP, pas annotation.** Webform 6.2 (compatible D10.2+/D11)
> remplace l'annotation Doctrine `@WebformHandler` par l'attribut PHP `#[WebformHandler]`
> (`Drupal\webform\Attribute\WebformHandler`). L'annotation reste tolérée mais est dépréciée — sur un
> projet D11 neuf, utiliser systématiquement l'attribut. Injecter le `http_client` via `create()` plutôt
> que `\Drupal::service()`.

```php
<?php

declare(strict_types=1);

// src/Plugin/WebformHandler/CrmSyncHandler.php
namespace Drupal\mon_module\Plugin\WebformHandler;

use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\webform\Attribute\WebformHandler;
use Drupal\webform\Plugin\WebformHandlerBase;
use Drupal\webform\Plugin\WebformHandlerInterface;
use Drupal\webform\WebformSubmissionInterface;
use GuzzleHttp\ClientInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Synchronise les soumissions avec le CRM.
 */
#[WebformHandler(
  id: 'mon_module_crm_sync',
  label: new TranslatableMarkup('Sync CRM'),
  category: new TranslatableMarkup('Mon Module'),
  description: new TranslatableMarkup('Envoie les données au CRM et crée un lead.'),
  cardinality: WebformHandlerInterface::CARDINALITY_SINGLE,
  results: WebformHandlerInterface::RESULTS_PROCESSED,
  submission: WebformHandlerInterface::SUBMISSION_REQUIRED,
)]
class CrmSyncHandler extends WebformHandlerBase {

  /**
   * Le client HTTP injecté.
   */
  protected ClientInterface $httpClient;

  /**
   * {@inheritdoc}
   *
   * Injection de dépendances — jamais \Drupal::service() dans la logique métier.
   */
  public static function create(ContainerInterface $container, array $configuration, $plugin_id, $plugin_definition): static {
    $instance = parent::create($container, $configuration, $plugin_id, $plugin_definition);
    $instance->httpClient = $container->get('http_client');
    return $instance;
  }

  /**
   * Formulaire de configuration du handler.
   */
  public function buildConfigurationForm(array $form, FormStateInterface $form_state): array {
    $form = parent::buildConfigurationForm($form, $form_state);

    $form['crm_api_url'] = [
      '#type' => 'textfield',
      '#title' => $this->t('CRM API URL'),
      '#default_value' => $this->configuration['crm_api_url'] ?? '',
      '#required' => TRUE,
    ];

    $form['crm_api_key'] = [
      '#type' => 'textfield',
      '#title' => $this->t('API Key'),
      '#default_value' => $this->configuration['crm_api_key'] ?? '',
    ];

    return $form;
  }

  /**
   * Déclenché lors de la soumission complète d'un formulaire.
   */
  public function postSave(WebformSubmissionInterface $webform_submission, bool $update = TRUE): void {
    if ($update) {
      return;  // Pas de resync si mise à jour d'une soumission existante
    }

    $data = $webform_submission->getData();
    $api_url = $this->configuration['crm_api_url'];
    $api_key = $this->configuration['crm_api_key'];

    // Construire le payload CRM
    $payload = [
      'first_name' => $data['prenom'] ?? '',
      'last_name'  => $data['nom'] ?? '',
      'email'      => $data['email'] ?? '',
      'source'     => 'drupal_webform',
      'form_id'    => $webform_submission->getWebform()->id(),
      'submission_id' => $webform_submission->id(),
    ];

    try {
      $response = $this->httpClient->post($api_url, [
        'json' => $payload,
        'headers' => ['Authorization' => 'Bearer ' . $api_key],
        'timeout' => 10,
      ]);

      if ($response->getStatusCode() >= 400) {
        $this->logger->error(
          'CRM sync failed for submission @id: HTTP @code',
          ['@id' => $webform_submission->id(), '@code' => $response->getStatusCode()]
        );
      }
      else {
        // Stocker le CRM ID dans les données de la soumission
        $crm_response = json_decode($response->getBody(), TRUE);
        $webform_submission->setElementData('crm_lead_id', $crm_response['id'] ?? '');
        $webform_submission->resave();
      }
    }
    catch (\Exception $e) {
      $this->logger->error(
        'CRM sync exception for submission @id: @error',
        ['@id' => $webform_submission->id(), '@error' => $e->getMessage()]
      );
    }
  }

  /**
   * Valider la configuration du handler.
   */
  public function validateConfigurationForm(array &$form, FormStateInterface $form_state): void {
    $url = $form_state->getValue('crm_api_url');
    if ($url && !filter_var($url, FILTER_VALIDATE_URL)) {
      $form_state->setError($form['crm_api_url'], $this->t('L\'URL CRM est invalide.'));
    }
  }

  public function defaultConfiguration(): array {
    return ['crm_api_url' => '', 'crm_api_key' => ''];
  }

  public function getSummary(): array {
    return [
      '#markup' => $this->t('Sync vers : @url', ['@url' => $this->configuration['crm_api_url']]),
    ];
  }
}
```

---

## Mode Test — Tester sans Envoyer

```
Webform → Settings → Test

Test mode :
  → Les handlers Email envoient à l'adresse de test (pas aux vraies adresses)
  → Les handlers Remote POST loggent au lieu d'envoyer
  → Utile pour valider la configuration avant la mise en production

Test depuis l'UI :
  Webform → Test → Remplir le formulaire de test → Soumettre
  → Vérifier les emails de test dans Maildev ou les logs

Test depuis Drush :
drush php:eval "
\$webform = \Drupal\webform\Entity\Webform::load('mon_formulaire');
\$submission = \Drupal\webform\Entity\WebformSubmission::create([
  'webform_id' => 'mon_formulaire',
  'data' => [
    'prenom' => 'Test',
    'email' => 'test@example.com',
    'message' => 'Test message',
  ],
]);
\$submission->save();
echo 'Soumission test créée : ' . \$submission->id();
"
```
