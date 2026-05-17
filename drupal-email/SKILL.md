---
name: drupal-email
description: Use when sending emails from Drupal with Symfony Mailer (drupal/symfony_mailer) - creating EmailBuilder plugins, writing Twig email templates (HTML + text versions), inlining CSS for email clients, configuring SMTP transport (Gmail, SendGrid, Mailgun, Amazon SES), testing emails locally with Maildev or Mailpit, replacing the legacy MailManager system, handling email translation, sending transactional emails programmatically, configuring the mail system for different operations (registration, password reset, notifications), debugging email delivery issues, or building a custom email notification system in Drupal 9-11+
---

# Drupal Email — Symfony Mailer — Référence Complète

## Overview

Référentiel complet de l'envoi d'emails Drupal 9-11+ avec le module `drupal/symfony_mailer` : EmailBuilder plugins, templates Twig HTML, CSS inlined, configuration SMTP (Gmail, SendGrid, Mailgun, SES), test local (Maildev/Mailpit), et migration depuis l'ancien système `MailManager`. Symfony Mailer remplace le système d'emails legacy de Drupal.

## 🎯 Symfony Mailer vs MailManager Legacy

```
Ancien système (MailManager + hook_mail) :
  ❌ Pas de templates HTML natifs (texte brut par défaut)
  ❌ Pas de transport SMTP configurable via UI
  ❌ CSS inline difficile
  ❌ Pas de preview

Symfony Mailer (drupal/symfony_mailer) :
  ✅ Templates Twig HTML natifs
  ✅ CSS inline automatique (Foundation Inky ou custom)
  ✅ Transport SMTP configurable via UI + variables env
  ✅ Compatibilité avec les anciens hook_mail via bridge
  ✅ Preview email dans l'admin
  ✅ Theming complet des emails par type
```

---

## Quick Decision Table

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Envoyer un email transactionnel simple | `symfony_mailer` → `EmailFactory` | [symfony-mailer-setup.md](symfony-mailer-setup.md) |
| Configurer SMTP (Gmail, SendGrid...) | Symfony Mailer → Transport → SMTP config | [smtp-config.md](smtp-config.md) |
| Template HTML pour un email | Twig template dans `templates/email/` | [email-templates.md](email-templates.md) |
| CSS inline dans les emails | `drupal/symfony_mailer` inliner automatique | [email-templates.md](email-templates.md) |
| Email de bienvenue utilisateur | Symfony Mailer → User → Welcome email | [symfony-mailer-setup.md](symfony-mailer-setup.md) |
| Email de reset password | Symfony Mailer → User → Password reset | [symfony-mailer-setup.md](symfony-mailer-setup.md) |
| Créer un EmailBuilder plugin | `@EmailBuilder` / `#[EmailBuilder]` D11 | [symfony-mailer-setup.md](symfony-mailer-setup.md) |
| Tester les emails en développement | Maildev (Docker) ou Mailpit | [email-testing.md](email-testing.md) |
| Capturer tous les emails en tests PHPUnit | `test_mail_collector` | [email-testing.md](email-testing.md) |
| Envoyer depuis PHP (programmatique) | `EmailFactory::newTypedEmail()` | [symfony-mailer-setup.md](symfony-mailer-setup.md) |
| Email multilingue (langue du destinataire) | `$email->setLangcode($langcode)` | [email-templates.md](email-templates.md) |
| SendGrid transport | `MAILER_DSN=sendgrid://API_KEY@default` | [smtp-config.md](smtp-config.md) |
| Amazon SES transport | `MAILER_DSN=ses+smtp://KEY:SECRET@default` | [smtp-config.md](smtp-config.md) |
| Mailgun transport | `MAILER_DSN=mailgun+api://KEY:DOMAIN@default` | [smtp-config.md](smtp-config.md) |
| Previewer les emails dans l'admin | Symfony Mailer → Preview | [email-testing.md](email-testing.md) |
| Migrer depuis hook_mail legacy | Symfony Mailer compatibility bridge | [symfony-mailer-setup.md](symfony-mailer-setup.md) |
| Configurer SPF (authorized senders) | DNS TXT `v=spf1 include:sendgrid.net ~all` | [email-deliverability.md](email-deliverability.md) |
| Configurer DKIM (signature emails) | DNS CNAME depuis le dashboard du provider SMTP | [email-deliverability.md](email-deliverability.md) |
| Configurer DMARC (politique compliance) | DNS TXT `_dmarc.mon-site.com` | [email-deliverability.md](email-deliverability.md) |
| Tester le score de délivrabilité | mail-tester.com, MX Toolbox | [email-deliverability.md](email-deliverability.md) |
| Gérer les bounces (emails invalides) | Return-Path + API provider SMTP | [email-deliverability.md](email-deliverability.md) |
| **Envoyer des emails en masse sans timeout** | Queue API + 1 email = 1 queue item + `QueueWorker` qui appelle `EmailFactory` | [symfony-mailer-setup.md](symfony-mailer-setup.md) |
| **Logger tous les emails envoyés** | EventSubscriber sur `MailerEvent::MESSAGE_SENT` → watchdog avec To + Subject | [email-testing.md](email-testing.md) |
| **Preview email dans l'UI admin** | Symfony Mailer → Settings → Policies → Preview button | [email-testing.md](email-testing.md) |
| **Email avec pièce jointe programmatique** | `$email->attachFromPath('/path/to/file.pdf', 'invoice.pdf')` | [symfony-mailer-setup.md](symfony-mailer-setup.md) |
| **Email inline image (logo dans le template)** | `$email->embed(File::fromPath($logo_path), 'logo')` → `<img src="{{ logo }}"` | [email-templates.md](email-templates.md) |
| **Compatibilité D8 — hook_mail legacy** | Symfony Mailer compatibility bridge — active automatiquement les anciens hook_mail | [symfony-mailer-setup.md](symfony-mailer-setup.md) |
| **Tester le score de délivrabilité avant go-live** | mail-tester.com — envoyer un email test et obtenir un score 10/10 | [email-deliverability.md](email-deliverability.md) |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| CSS externe (`<link>`) dans un email HTML | CSS inliné (Symfony Mailer le fait automatiquement) | Gmail, Outlook ignorent les CSS externes |
| Emails en texte brut seulement | Template HTML + fallback texte | CTR réduit, pas de mise en forme |
| SMTP credentials en clair dans settings.php | Variable d'environnement `MAILER_DSN` | Credentials exposés dans git |
| `mb_send_mail()` natif PHP | `symfony_mailer` pour la compatibilité | Pas de template, pas de tracking, pas de retry |
| Pas de test d'email avant mise en prod | Maildev en dev + test_mail_collector en PHPUnit | Emails cassés découverts en prod |
| Email sans version texte | Toujours inclure `{text}` block dans le template | Clients email texte seul (certains gouvernements) |

## Évolution par Version Majeure

| Feature | D9 | D10 | D11 |
|---------|----|----|-----|
| Symfony Mailer (contrib) | ✅ | ✅ | ✅ |
| MailManager legacy | ✅ | ✅ | ⚠️ |
| `hook_mail()` | ✅ | ✅ | ✅ |
| EmailBuilder plugin | ✅ | ✅ | ✅ |
| `#[EmailBuilder]` attribute | ❌ | ✅ opt. | ✅ std. |
| DSN transport configuration | ✅ | ✅ | ✅ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Problèmes d'envoi email découverts en projet.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions.

## See Also

- `drupal-core` — MailManager legacy, `hook_mail()`, service `plugin.manager.mail`
- `drupal-theming` — Templates email Twig, CSS inline, email-templates.md
- `drupal-webform` — Handlers Email Webform
- `drupal-docker` — Maildev/Mailpit comme service Docker
- `drupal-security` — SPF/DKIM, credentials SMTP
