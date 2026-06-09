# Changelog — drupal-email

---

## v1.2 — 2026-06-08 (correctifs API Symfony Mailer D11)

**Corrections d'exactitude technique :**
- `createParams()` : signature variadique conforme à `EmailBuilderBase` (était re-typée → fatal au `drush cr`) — `symfony-mailer-setup.md`
- `importTransportConfig()` : méthode inexistante remplacée par le flux réel (activation module + policies via UI/config) — `symfony-mailer-setup.md`
- `setReturnPath()` : méthode absente d'`EmailInterface` ; remplacée par l'accès au Mime Email via `getInner()` dans un EmailProcessor + note sur la réécriture par le provider — `email-deliverability.md`
- Test PHPUnit : suppression de la fausse clé `symfony_mailer.settings.test_transport`, conservation du seul `test_mail_collector` — `email-testing.md`
- Logging emails : `MailerEvent::MESSAGE_SENT` (inexistant) → `MailerSendEvent` / EmailProcessor phase `POST_SEND` — `SKILL.md`

**Complétude :**
- Statut explicite du legacy MailManager (legacy, non supprimé en D11, bridge `symfony_mailer_bc`) — `SKILL.md`
- 3 nouvelles leçons (signature createParams, Return-Path, méthode inventée) — `lessons.md`

---

## v1.1 — 2026-05-16 (audit complet)

**Corrections :**
- See Also mis à jour (drupal-tooling remplacé par drupal-deployment)
- Leçons enrichies (5 leçons au total)
- Fichiers manquants créés (liens QDT résolus)

---

## v1.0 — 2026-05-16

**Création initiale**

- SKILL.md avec Quick Decision Table (4 fichiers de référence)
- lessons.md avec incidents réels
