---
name: drupal-webform
description: Use when building forms with Drupal Webform module - configuring form elements (text, select, checkboxes, date, file upload, address, composite), implementing conditional logic with #states, creating multi-step wizard forms, configuring email notification handlers, saving submissions to database, accessing Webform submissions programmatically (WebformSubmission entity), implementing Remote POST handlers for CRM/webhook integration, creating custom WebformHandler plugins, exporting submissions as CSV, configuring per-element and per-form access control, adding RGPD consent checkboxes, pre-filling form fields from URL query parameters or user data, configuring Webform API REST endpoints, or debugging form validation errors in Drupal 8-11+
---

# Drupal Webform — Référence Complète

## Overview

Référentiel complet du module Webform Drupal 8-11+ : éléments de formulaire (50+ types), logique conditionnelle, formulaires multi-étapes, handlers (email, remote post, action), soumissions programmatiques, API REST, et handlers custom. Webform est le module de formulaires de référence pour Drupal.

## 🎯 La Règle Fondamentale

> **Webform avant Forms API.** Si un formulaire est destiné aux éditeurs ou peut évoluer, utiliser Webform (configurable via UI). Réserver la Forms API pour les formulaires d'administration liés au code du module.

---

## Quick Decision Table

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Formulaire de contact simple | Webform UI — ajouter les éléments | [webform-elements.md](webform-elements.md) |
| Champ texte, email, téléphone | Elements : textfield, email, tel | [webform-elements.md](webform-elements.md) |
| Liste déroulante, radio, checkboxes | Elements : select, radios, checkboxes | [webform-elements.md](webform-elements.md) |
| Upload de fichier dans un formulaire | Element : managed_file + config | [webform-elements.md](webform-elements.md) |
| Champ adresse complète (structure) | Element : address (contrib) ou composite | [webform-elements.md](webform-elements.md) |
| Champ conditionnel (visible si X = Y) | `#states` dans la config de l'élément | [webform-elements.md](webform-elements.md) |
| Formulaire en plusieurs étapes (wizard) | Webform → wizard → pages | [webform-elements.md](webform-elements.md) |
| Email de confirmation à l'utilisateur | Handler Email → "To: [webform_submission:values:email]" | [webform-handlers.md](webform-handlers.md) |
| Email de notification à l'admin | Handler Email → To: admin@site.com | [webform-handlers.md](webform-handlers.md) |
| Envoyer les données à un CRM (POST) | Handler Remote POST → JSON | [webform-handlers.md](webform-handlers.md) |
| Webhook sur soumission | Handler Remote POST | [webform-handlers.md](webform-handlers.md) |
| Handler custom (logique métier) | `WebformHandlerBase` plugin | [webform-handlers.md](webform-handlers.md) |
| Enregistrer les soumissions en DB | Settings → Store submissions : ✅ | [webform-submissions.md](webform-submissions.md) |
| Lire une soumission depuis PHP | `WebformSubmission::load()` | [webform-submissions.md](webform-submissions.md) |
| Créer une soumission programmatiquement | `WebformSubmission::create()` | [webform-submissions.md](webform-submissions.md) |
| Exporter les soumissions en CSV | Results → Download → CSV | [webform-submissions.md](webform-submissions.md) |
| Afficher les résultats en table | Results → Results table | [webform-submissions.md](webform-submissions.md) |
| Accès au formulaire par rôle | Access → Create submissions | [webform-access.md](webform-access.md) |
| Afficher le formulaire uniquement aux connectés | Access → Authenticated users only | [webform-access.md](webform-access.md) |
| Limiter une soumission par utilisateur | Settings → Limit → 1 par utilisateur | [webform-access.md](webform-access.md) |
| Consentement RGPD (checkbox obligatoire) | Element : checkbox + required | [webform-elements.md](webform-elements.md) |
| Pré-remplir depuis l'URL (`?name=Jean`) | Default value → token ou query | [webform-elements.md](webform-elements.md) |
| Endpoint REST pour soumettre via API | Webform REST → POST endpoint | [webform-api.md](webform-api.md) |
| Tester les emails sans les envoyer | Webform → Settings → Test mode | [webform-handlers.md](webform-handlers.md) |
| Visualiser les formulaires soumis | Results → Submissions | [webform-submissions.md](webform-submissions.md) |
| **Anti-spam : Honeypot (invisible)** | Module `honeypot` + Webform settings → Spam protection | [webform-access.md](webform-access.md) |
| **Anti-spam : CAPTCHA invisible** | Module `recaptcha` (v2 invisible ou v3) + Webform element | [webform-access.md](webform-access.md) |
| Anti-spam : taux de soumission (rate limiting) | Module `webform_spam_control` ou `flood` core | [webform-access.md](webform-access.md) |
| **Élément calculé (valeur dépend d'autres champs)** | Webform element `computed_twig` — `{{ data.prenom }} {{ data.nom }}` | [webform-elements.md](webform-elements.md) |
| **Afficher les soumissions Webform dans une View** | `drupal/webform_views` — expose les soumissions comme source Views | [webform-submissions.md](webform-submissions.md) |
| **Modifier le comportement du formulaire selon l'URL** | `[current-page:query:source]` token + default value conditionnel | [webform-elements.md](webform-elements.md) |
| **Envoyer un PDF de confirmation** | Handler Email + pièce jointe générée avec `drupal/webform_entity_print` | [webform-handlers.md](webform-handlers.md) |
| **Formulaire attaché à un nœud (1 form par article)** | Webform field sur le Content Type → une instance par nœud | [webform-api.md](webform-api.md) |
| **Handler conditionnel (envoyer email seulement si type=X)** | Webform Handler → Conditions tab → `data[type] = 'pro'` | [webform-handlers.md](webform-handlers.md) |
| **Déboguer une validation échouée** | `drush webform:submission:export WEBFORM_ID --format=table` + vérifier les errors | [webform-submissions.md](webform-submissions.md) |
| **Webform accessible (WCAG 2.1 AA)** | Labels explicites, `aria-describedby` sur les descriptions, gestion du focus en multi-step | [webform-elements.md](webform-elements.md) |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| Formulaire de contact codé en Forms API | Webform module — configurable sans déploiement | Les éditeurs ne peuvent pas modifier un formulaire codé |
| Stocker des données sensibles sans chiffrement | Champ Webform avec `#access: false` pour les admins + chiffrement DB | Data breach |
| Email handler sans test préalable | Webform → Settings → Test mode → envoyer un test | Emails de confirmation cassés en production |
| Remote POST sans vérification de la réponse | Vérifier le status code HTTP dans le handler | Soumissions perdues silencieusement |
| Validation custom en JavaScript uniquement | Toujours valider côté serveur (validator element) | Bypassable |
| Webform avec `store: false` pour des données critiques | Toujours stocker + configurer la rétention | Pas de récupération en cas de crash email |
| Pré-remplissage depuis `$_GET` sans sanitisation | Utiliser les tokens Webform sécurisés | XSS via URL query |
| CAPTCHA absent sur un formulaire public | Module `webform_spam_control` ou Honeypot | Spam massif |

## Évolution par Version Majeure

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| Webform module | contrib | contrib | contrib | contrib |
| Wizard (multi-step) | ✅ | ✅ | ✅ | ✅ |
| Remote POST handler | ✅ | ✅ | ✅ | ✅ |
| Webform REST API | ✅ | ✅ | ✅ | ✅ |
| YAML import/export | ✅ | ✅ | ✅ | ✅ |
| Computed elements | ✅ | ✅ | ✅ | ✅ |
| Custom handlers | ✅ | ✅ | ✅ | ✅ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Problèmes Webform découverts en projet.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions du skill.

## See Also

- `drupal-core` — Forms API (quand Webform ne suffit pas), services DI
- `drupal-security` — Validation XSS, fichiers uploadés, CSRF protection
- `drupal-migration` — Migrer des soumissions Webform
- `drupal-api` — REST endpoint Webform pour les applications headless
- `drupal-content-modeling` — Webform attaché à un Content Type (node)
- `drupal-config` — Export/import config Webform (YAML)
