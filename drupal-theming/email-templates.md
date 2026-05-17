# Templates Email — Symfony Mailer & Drupal 11

## Structure des templates email

```
mon_module/
└── templates/
    └── email/
        ├── email-confirmation.html.twig    # Confirmation de compte
        ├── email-notification.html.twig    # Notification générique
        └── email-password-reset.html.twig  # Réinitialisation mot de passe
```

> Pas d'héritage (`extends`) dans les templates email — chaque email est autonome avec sa propre structure HTML complète.

---

## Template email de base

```twig
{# templates/email/email-confirmation.html.twig #}
<!DOCTYPE html>
<html lang="{{ language.id }}" dir="{{ language.direction }}">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <title>{{ subject }}</title>
  <style>
    /* CSS inline OBLIGATOIRE — les clients email (Gmail, Outlook) ignorent les feuilles externes */
    * { box-sizing: border-box; }
    body {
      font-family: Arial, Helvetica, sans-serif;
      line-height: 1.6;
      color: #333333;
      margin: 0;
      padding: 0;
      background-color: #f4f4f4;
    }
    .wrapper { width: 100%; padding: 20px 0; background-color: #f4f4f4; }
    .container {
      max-width: 600px;
      margin: 0 auto;
      background-color: #ffffff;
      border-radius: 4px;
      overflow: hidden;
    }
    .header {
      background-color: #0071b8;
      padding: 24px 32px;
      text-align: center;
    }
    .header h1 { color: #ffffff; margin: 0; font-size: 24px; }
    .content { padding: 32px; }
    .button-wrapper { text-align: center; margin: 32px 0; }
    .button {
      display: inline-block;
      background-color: #0071b8;
      color: #ffffff !important;  /* !important nécessaire pour certains clients email */
      padding: 14px 28px;
      text-decoration: none;
      border-radius: 4px;
      font-weight: bold;
      font-size: 16px;
    }
    .footer {
      padding: 16px 32px;
      background-color: #f8f8f8;
      border-top: 1px solid #e0e0e0;
      font-size: 12px;
      color: #666666;
      text-align: center;
    }
    /* Responsive — support limité selon les clients email */
    @media only screen and (max-width: 600px) {
      .container { width: 100% !important; }
      .content { padding: 16px !important; }
    }
  </style>
</head>
<body>
  <div class="wrapper">
    <div class="container">
      <div class="header">
        <h1>{{ site_name }}</h1>
      </div>
      <div class="content">
        <p>{{ 'Bonjour @name,'|t({'@name': user.displayName}) }}</p>
        <p>{{ 'Merci de vous être inscrit sur @site. Confirmez votre adresse email pour activer votre compte.'|t({'@site': site_name}) }}</p>
        <div class="button-wrapper">
          <a href="{{ confirmation_url }}" class="button">
            {{ 'Confirmer mon compte'|t }}
          </a>
        </div>
        <p>
          <small>{{ 'Ce lien expire dans 24 heures.'|t }}</small><br>
          <small>{{ 'Si vous n\'avez pas créé de compte, ignorez cet email.'|t }}</small>
        </p>
        <p>
          {{ 'Ou copiez ce lien dans votre navigateur :'|t }}<br>
          <small><a href="{{ confirmation_url }}">{{ confirmation_url }}</a></small>
        </p>
      </div>
      <div class="footer">
        <p>{{ '@site — Cet email a été envoyé à @email'|t({'@site': site_name, '@email': user.mail}) }}</p>
        <p><a href="{{ site_url }}">{{ site_url }}</a></p>
      </div>
    </div>
  </div>
</body>
</html>
```

---

## Enregistrer le template dans le module

```php
// mon_module.module

/**
 * Implements hook_theme().
 */
function mon_module_theme(): array {
  return [
    'email_confirmation' => [
      'variables' => [
        'user'             => NULL,
        'confirmation_url' => NULL,
        'site_name'        => NULL,
        'site_url'         => NULL,
        'subject'          => '',
        'language'         => NULL,
      ],
    ],
    'email_notification' => [
      'variables' => [
        'user'        => NULL,
        'message'     => '',
        'action_url'  => NULL,
        'action_label'=> NULL,
        'site_name'   => NULL,
        'subject'     => '',
        'language'    => NULL,
      ],
    ],
  ];
}
```

---

## Envoyer l'email depuis un service ou contrôleur

```php
// src/Service/UserNotificationService.php

use Drupal\Core\Language\LanguageManagerInterface;
use Drupal\Core\Mail\MailManagerInterface;
use Drupal\Core\Render\RendererInterface;
use Drupal\user\UserInterface;

class UserNotificationService {

  public function __construct(
    private readonly MailManagerInterface $mailManager,
    private readonly RendererInterface $renderer,
    private readonly LanguageManagerInterface $languageManager,
  ) {}

  public function sendConfirmation(UserInterface $user, string $confirmationUrl): void {
    $language = $this->languageManager->getLanguage($user->getPreferredLangcode());

    $build = [
      '#theme'            => 'email_confirmation',
      '#user'             => $user,
      '#confirmation_url' => $confirmationUrl,
      '#site_name'        => \Drupal::config('system.site')->get('name'),
      '#site_url'         => \Drupal::request()->getSchemeAndHttpHost(),
      '#subject'          => t('Confirmez votre compte', [], ['langcode' => $user->getPreferredLangcode()]),
      '#language'         => $language,
    ];

    $this->mailManager->mail(
      module: 'mon_module',
      key: 'confirmation',
      to: $user->getEmail(),
      langcode: $user->getPreferredLangcode(),
      params: [
        'subject' => t('Confirmez votre compte sur @site', [
          '@site' => \Drupal::config('system.site')->get('name'),
        ], ['langcode' => $user->getPreferredLangcode()]),
        'body' => $this->renderer->renderInIsolation($build),
      ],
    );
  }
}
```

---

## Implémenter hook_mail() pour les paramètres

```php
// mon_module.module

/**
 * Implements hook_mail().
 */
function mon_module_mail(string $key, array &$message, array $params): void {
  switch ($key) {
    case 'confirmation':
      $message['subject'] = $params['subject'];
      $message['body'][]  = $params['body'];
      // Indiquer que le body est du HTML
      $message['headers']['Content-Type'] = 'text/html; charset=UTF-8';
      break;

    case 'notification':
      $message['subject'] = $params['subject'];
      $message['body'][]  = $params['body'];
      $message['headers']['Content-Type'] = 'text/html; charset=UTF-8';
      break;
  }
}
```

---

## Tester les emails avec MailHog

```yaml
# docker-compose.yml — ajouter le service mailhog
services:
  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "1025:1025"   # SMTP
      - "8025:8025"   # Interface web
```

```php
// settings.local.php — rediriger vers MailHog en dev
$config['symfony_mailer.mailer_transport.smtp']['configuration']['host'] = 'mailhog';
$config['symfony_mailer.mailer_transport.smtp']['configuration']['port'] = '1025';
$config['symfony_mailer.mailer_transport.smtp']['configuration']['user'] = '';
$config['symfony_mailer.mailer_transport.smtp']['configuration']['pass'] = '';

// Ou avec le module Reroute Email (contrib) :
$config['reroute_email.settings']['enable'] = TRUE;
$config['reroute_email.settings']['address'] = 'dev@localhost';
```

```bash
# Accéder à l'interface MailHog
# http://localhost:8025

# Tester l'envoi depuis le container
docker compose exec php drush php:eval "
  \$mail = \Drupal::service('plugin.manager.mail');
  \$mail->mail('mon_module', 'confirmation', 'test@example.com', 'fr', ['subject' => 'Test', 'body' => '<p>Test</p>']);
"
```

---

## Anti-patterns templates email

| ❌ À éviter | ✅ Bonne pratique | Raison |
|------------|------------------|--------|
| CSS dans `<link>` externe | CSS inline dans `<style>` ou attribut `style=""` | Gmail supprime les `<link>` externes |
| Flexbox / Grid CSS | Tables pour la mise en page (ou CSS simple) | Outlook ne supporte pas Flexbox/Grid |
| `background-image:` CSS | `<td background="...">` ou image `<img>` | Support client email limité |
| Images sans `alt` | Attribut `alt` sur toutes les `<img>` | Gmail bloque les images par défaut |
| URL relative (`/chemin`) | URL absolue (`https://monsite.com/chemin`) | L'email n'a pas de base URL |
| `{{ confirmation_url\|raw }}` | Auto-escape Twig (par défaut) | XSS si l'URL est manipulée |
| `extends` Twig dans email | Chaque template email est autonome | Les emails nécessitent HTML complet |
