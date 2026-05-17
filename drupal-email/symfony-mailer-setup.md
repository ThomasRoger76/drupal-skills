---
name: drupal-email — symfony mailer setup
description: Installer et configurer drupal/symfony_mailer - EmailBuilder plugin, envoi programmatique, et configuration du transport.
---

# Symfony Mailer Setup — Configuration Complète

## Installation

```bash
composer require drupal/symfony_mailer
drush en symfony_mailer -y

# Importer les policies par défaut (emails système)
drush php:eval "
foreach (\Drupal::service('plugin.manager.email_builder')->getDefinitions() as \$id => \$def) {
  echo \$id . PHP_EOL;
}
"
```

---

## EmailBuilder Plugin — Envoyer un Email Typé

```php
<?php
// src/Plugin/EmailBuilder/CommandeEmailBuilder.php
namespace Drupal\mon_module\Plugin\EmailBuilder;

use Drupal\symfony_mailer\EmailBuilderBase;
use Drupal\symfony_mailer\EmailInterface;
use Drupal\symfony_mailer\Processor\EmailProcessorBase;
use Drupal\mon_module\Entity\Commande;

/**
 * @EmailBuilder(
 *   id = "mon_module",
 *   label = @Translation("Mon Module"),
 *   sub_types = {
 *     "commande_confirmee" = @Translation("Confirmation de commande"),
 *     "commande_expediee" = @Translation("Commande expédiée"),
 *   }
 * )
 */
// D11 : #[EmailBuilder(...)]
class CommandeEmailBuilder extends EmailBuilderBase {

  /**
   * Initialiser l'email depuis les paramètres passés.
   */
  public function createParams(EmailInterface $email, ?Commande $commande = NULL): void {
    if ($commande) {
      $email->setParam('commande', $commande);
    }
  }

  /**
   * Construire l'email — appelé au moment de l'envoi.
   */
  public function build(EmailInterface $email): void {
    $commande = $email->getParam('commande');
    $sub_type = $email->getSubType();

    // Destinataire
    $user = $commande->get('uid')->entity;
    $email->setTo($user->getEmail(), $user->getDisplayName());

    // Sujet selon le sous-type
    $subjects = [
      'commande_confirmee' => $this->t('Commande #[commande:reference] confirmée'),
      'commande_expediee' => $this->t('Commande #[commande:reference] expédiée'),
    ];
    $email->setSubject((string) $subjects[$sub_type]);

    // Variables disponibles dans le template Twig
    $email->setVariable('commande', $commande);
    $email->setVariable('user', $user);
    $email->setVariable('sub_type', $sub_type);
  }
}
```

---

## Envoyer un Email Programmatiquement

```php
// Envoyer depuis n'importe quel code Drupal
use Drupal\symfony_mailer\EmailFactoryInterface;

// Via injection de dépendances (recommandé)
class CommandeService {
  public function __construct(
    private readonly EmailFactoryInterface $emailFactory,
  ) {}

  public function envoyerConfirmation(Commande $commande): void {
    $this->emailFactory
      ->newTypedEmail('mon_module', 'commande_confirmee', $commande)
      ->send();
  }
}

// Via service statique (moins recommandé)
\Drupal::service('email_factory')
  ->newTypedEmail('mon_module', 'commande_confirmee', $commande)
  ->send();

// Email simple sans EmailBuilder
\Drupal::service('email_factory')
  ->newEmail()
  ->setTo('destinataire@example.com')
  ->setSubject('Test email')
  ->setHtmlBody('<p>Bonjour depuis Drupal !</p>')
  ->setTextBody('Bonjour depuis Drupal !')
  ->send();
```

---

## Template Twig de l'Email

```twig
{# templates/email/mon_module--commande-confirmee.html.twig #}
{# Ce template est découvert automatiquement par Symfony Mailer #}

{% extends "@symfony_mailer/base.html.twig" %}

{% block subject %}
  Confirmation de votre commande {{ commande.reference.value }}
{% endblock %}

{% block body %}
<table style="width:100%; max-width:600px; margin:0 auto; font-family: Arial, sans-serif;">
  <tr>
    <td style="background:#1a5276; padding:20px; text-align:center;">
      <img src="{{ logo_url }}" alt="{{ site_name }}" height="50">
    </td>
  </tr>
  <tr>
    <td style="padding:30px;">
      <h1 style="color:#1a5276;">Commande confirmée !</h1>
      <p>Bonjour {{ user.displayname }},</p>
      <p>Votre commande <strong>{{ commande.reference.value }}</strong> a bien été reçue.</p>

      <table style="width:100%; border-collapse:collapse; margin:20px 0;">
        <tr style="background:#f8f9fa;">
          <th style="padding:10px; text-align:left;">Détail</th>
          <th style="padding:10px; text-align:right;">Montant</th>
        </tr>
        <tr>
          <td style="padding:10px; border-bottom:1px solid #dee2e6;">Total</td>
          <td style="padding:10px; border-bottom:1px solid #dee2e6; text-align:right;">
            {{ commande.montant.value|number_format(2, ',', ' ') }} €
          </td>
        </tr>
      </table>

      <p>
        <a href="{{ url('entity.commande.canonical', {commande: commande.id}) }}"
           style="background:#1a5276; color:#fff; padding:12px 24px; text-decoration:none; border-radius:4px;">
          Voir ma commande
        </a>
      </p>
    </td>
  </tr>
  <tr>
    <td style="background:#f8f9fa; padding:15px; text-align:center; font-size:12px; color:#666;">
      © {{ "now"|date("Y") }} {{ site_name }} — {{ site_url }}
    </td>
  </tr>
</table>
{% endblock %}
```

---

## Remplacement des Emails Système Drupal

```bash
# Importer les policies par défaut (emails user, password reset, etc.)
drush php:eval "\Drupal::service('symfony_mailer.helper')->importTransportConfig();"

# Vérifier les policies actives
# /admin/config/system/mailer/policy
```

```
/admin/config/system/mailer/policy

Policies disponibles :
  ├── User — Welcome (registered)       → email de bienvenue
  ├── User — Password reset             → lien de réinitialisation
  ├── User — Email verification         → confirmation email
  └── Contact — Site contact form       → formulaire de contact Drupal
```
