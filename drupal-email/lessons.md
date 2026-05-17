# Leçons — drupal-email

Problèmes d'emails Drupal découverts en projet réel.

---

### 2026-05-16 — CSS non rendu dans Gmail / Outlook — CSS externe

- **Symptôme :** Emails sans mise en forme dans Gmail et Outlook malgré un beau template
- **Cause :** CSS dans `<style>` ou `<link>` externe — Gmail et Outlook ignorent le CSS externe
- **Correct :** Symfony Mailer inline le CSS automatiquement si correctement configuré. Vérifier que `inline_css` est activé dans la policy.
- **Prévention :** Toujours tester les emails dans Litmus ou Email on Acid avant la mise en production

### 2026-05-16 — Credentials SMTP en clair dans settings.php commité

- **Symptôme :** Credentials SendGrid visibles dans l'historique git
- **Cause :** `$config['symfony_mailer...']['dsn'] = 'sendgrid://API_KEY@default'` hardcodé
- **Correct :** `$config[...]['dsn'] = getenv('MAILER_DSN')` + variable d'environnement CI/CD
- **Prévention :** Jamais de credentials en clair dans settings.php. Toujours `getenv()`.

### 2026-05-16 — Emails de test envoyés en production depuis l'environnement de staging

- **Symptôme :** Des clients ont reçu des emails de test
- **Cause :** Staging pointait vers le transport SMTP production au lieu de Maildev
- **Correct :** `MAILER_DSN=smtp://maildev:1025` sur staging. SMTP réel uniquement sur prod.
- **Prévention :** Toujours Maildev/Null transport sur staging. Vérifier la variable `MAILER_DSN` avant tout déploiement.

### 2026-05-16 — EmailBuilder non découvert — plugin manquant

- **Symptôme :** `Plugin 'mon_module' not found` lors de l'envoi d'un email
- **Cause :** `@EmailBuilder` annotation ou `#[EmailBuilder]` attribute incorrect / dans le mauvais namespace
- **Correct :** Vérifier le namespace `Drupal\mon_module\Plugin\EmailBuilder\` + `drush cr`
- **Prévention :** `drush cr` après création de tout EmailBuilder. Vérifier le namespace exact.

### 2026-05-16 — Email de test jamais capturé dans PHPUnit

- **Symptôme :** `\Drupal::state()->get('system.test_mail_collector')` retourne NULL dans le test
- **Cause :** Le collecteur de test n'était pas configuré avant l'action qui déclenche l'email
- **Correct :** Configurer dans `setUp()` : `\Drupal::configFactory()->getEditable('system.mail')->set('interface.default', 'test_mail_collector')->save();`
- **Prévention :** Pattern systématique : configurer le collecteur en PREMIER dans `setUp()`, avant tout autre code

### 2026-05-16 — Images cassées dans Outlook — chemins relatifs

- **Symptôme :** Images visibles dans Gmail, cassées dans Outlook
- **Cause :** URLs d'images relatives (`/sites/default/files/...`) au lieu d'URLs absolues
- **Correct :** `{{ absolute_url(image_url) }}` dans le template Twig email
- **Prévention :** Règle : toutes les URLs d'images dans les emails doivent être absolues

### 2026-05-16 — Email envoyé depuis le mauvais expéditeur

- **Symptôme :** FROM = `admin@site.com` au lieu de `noreply@site.com`
- **Cause :** FROM non configuré dans l'EmailBuilder → utilise le FROM global Drupal (email admin)
- **Correct :** `$email->setFrom('noreply@site.com', 'Mon Site');` dans EmailBuilder::build()
- **Prévention :** Configurer le FROM globalement dans `/admin/config/system/mailer` ET vérifier par EmailBuilder
