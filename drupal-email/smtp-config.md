---
name: drupal-email — SMTP configuration
description: Configurer les transports SMTP dans Symfony Mailer Drupal - Gmail, SendGrid, Mailgun, Amazon SES, et variables d'environnement DSN.
---

# SMTP Configuration — Tous les Transporteurs

## Configuration via `MAILER_DSN` (Recommandé)

```php
// settings.php — définir le transport via DSN
$config['symfony_mailer.mailer_transport.default']['configuration']['dsn'] = getenv('MAILER_DSN');

// OU settings.php prod (selon la plateforme)
// La variable MAILER_DSN est configurée en variable d'environnement CI/CD
```

---

## DSN par Service

### Gmail

```bash
# Format DSN Gmail
MAILER_DSN=gmail://EMAIL:PASSWORD@default
# OU avec App Password (2FA)
MAILER_DSN=gmail://EMAIL:APP_PASSWORD@default

# Configurer "App Password" Google :
# → myaccount.google.com → Sécurité → Mots de passe des applications
```

### SendGrid

```bash
# Compte SendGrid — SMTP
MAILER_DSN=smtp://apikey:SENDGRID_API_KEY@smtp.sendgrid.net:587

# OU SendGrid API
MAILER_DSN=sendgrid://SENDGRID_API_KEY@default
```

### Mailgun

```bash
# Mailgun SMTP
MAILER_DSN=smtp://postmaster@DOMAIN:SMTP_PASSWORD@smtp.mailgun.org:587

# OU Mailgun API
MAILER_DSN=mailgun+api://KEY:DOMAIN@default
# OU region EU
MAILER_DSN=mailgun+api://KEY:DOMAIN@default?region=eu
```

### Amazon SES

```bash
# SES SMTP (région us-east-1)
MAILER_DSN=smtp://AWS_ACCESS_KEY:AWS_SECRET_KEY@email-smtp.us-east-1.amazonaws.com:587

# OU SES API
MAILER_DSN=ses+smtp://AWS_ACCESS_KEY:AWS_SECRET_KEY@default?region=eu-west-1
```

### SMTP Générique

```bash
# SMTP standard (ex: Infomaniak, OVH, Mailjet...)
MAILER_DSN=smtp://USERNAME:PASSWORD@smtp.example.com:587
# Avec chiffrement TLS
MAILER_DSN=smtp://USERNAME:PASSWORD@smtp.example.com:587?encryption=tls&auth_mode=login
```

### Maildev / Mailpit (Développement)

```bash
# Maildev — capturer tous les emails en dev
MAILER_DSN=smtp://maildev:1025
# OU
MAILER_DSN=smtp://mailpit:1025

# Service Docker dans docker-compose.yml
maildev:
  image: maildev/maildev
  ports:
    - "${MAILDEV_PORT:-83}:1080"   # Interface web
    # Port SMTP 1025 reste interne au réseau Docker
```

---

## Configuration via l'UI Symfony Mailer

```
/admin/config/system/mailer/transport/default

Types de transport :
  ├── SMTP         → serveur SMTP externe
  ├── Native       → PHP mail() (non recommandé)
  ├── Null         → email capturés, jamais envoyés (tests)
  └── Sendmail     → sendmail local
```

---

## Tester la Configuration

```bash
# Envoyer un email de test depuis Drush
drush php:eval "
\Drupal::service('email_factory')
  ->newEmail()
  ->setTo('test@example.com', 'Test')
  ->setSubject('Test Symfony Mailer — ' . date('Y-m-d H:i'))
  ->setHtmlBody('<p>Email de test depuis Drupal.</p>')
  ->setTextBody('Email de test depuis Drupal.')
  ->send();
echo 'Email envoyé.';
"

# Vérifier les logs
drush watchdog:show --type=symfony_mailer --count=10
```

---

## Variables d'Environnement par Plateforme

```bash
# Platform.sh
platform variable:set MAILER_DSN "sendgrid://API_KEY@default" --sensitive -p PROJECT_ID -e main

# Pantheon
terminus secret:set SITE.ENVIRONMENT MAILER_DSN "smtp://user:pass@smtp.example.com:587"

# Docker Compose
# .env (gitignored)
MAILER_DSN=smtp://maildev:1025

# GitLab CI/CD
# Settings → CI/CD → Variables → MAILER_DSN (masked)

# Acquia
acli api:environments:variables:create ENV_ID --name=MAILER_DSN --value="..." --is-sensitive=1
```

---

## From Address — Expéditeur par Défaut

```php
// settings.php — expéditeur par défaut
$config['symfony_mailer.settings']['default_from']['address'] = getenv('MAIL_FROM') ?: 'noreply@mon-site.com';
$config['symfony_mailer.settings']['default_from']['name'] = getenv('SITE_NAME') ?: 'Mon Site';
```

```
/admin/config/system/mailer

→ Default "from" address : noreply@mon-site.com
→ Default "from" name : Mon Site
```
