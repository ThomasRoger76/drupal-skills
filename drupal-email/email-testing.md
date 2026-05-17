---
name: drupal-email — testing
description: Tester les emails Drupal en développement et dans les tests PHPUnit - Maildev, Mailpit, test_mail_collector, preview Symfony Mailer.
---

# Email Testing — Référence Complète

## Maildev — Capturer les Emails en Dev

```yaml
# docker-compose.yml
maildev:
  image: maildev/maildev
  ports:
    - "${MAILDEV_PORT:-83}:1080"    # Interface web : http://localhost:83
    # SMTP port 1025 reste interne au réseau Docker
  environment:
    MAILDEV_WEB_PORT: 1080
    MAILDEV_SMTP_PORT: 1025
```

```bash
# Transport SMTP pointant vers Maildev
# .env (dev)
MAILER_DSN=smtp://maildev:1025

# Accéder aux emails capturés
# → http://localhost:83 (interface web Maildev)

# Via Drush — voir les emails capturés
docker compose exec maildev sh -c "ls /mail/*.json" 2>/dev/null || echo "Pas de mails"
```

---

## Mailpit — Alternative Moderne à Maildev

```yaml
# docker-compose.yml
mailpit:
  image: axllent/mailpit
  ports:
    - "${MAILPIT_PORT:-83}:8025"    # Interface web (8025 interne)
    # SMTP 1025 interne au réseau Docker
```

```bash
# Transport SMTP Mailpit
MAILER_DSN=smtp://mailpit:1025

# Interface web → http://localhost:83
# Mailpit supporte la recherche et l'API REST
```

---

## test_mail_collector — Tests PHPUnit

```php
// Dans KernelTestBase ou BrowserTestBase — configurer le collecteur d'emails

class CommandeEmailTest extends \Drupal\Tests\BrowserTestBase {

  protected static $modules = ['symfony_mailer', 'mon_module'];
  protected $defaultTheme = 'stark';

  protected function setUp(): void {
    parent::setUp();

    // Activer le collecteur de test (emails capturés en mémoire, jamais envoyés)
    $this->config('symfony_mailer.settings')
      ->set('test_transport', TRUE)
      ->save();

    // OU via test_mail_collector (système legacy)
    $this->config('system.mail')
      ->set('interface.default', 'test_mail_collector')
      ->save();
  }

  public function testConfirmationEmail(): void {
    // Créer une commande qui déclenche l'email
    $commande = $this->createCommande(['statut' => 'pending']);
    $commande->set('statut', 'confirmed');
    $commande->save();  // Déclenche l'email de confirmation

    // Récupérer les emails capturés
    $emails = \Drupal::state()->get('system.test_mail_collector') ?? [];
    $this->assertCount(1, $emails, 'Un email de confirmation a été envoyé.');

    $email = reset($emails);
    $this->assertStringContainsString($commande->get('reference')->value, $email['subject']);
    $this->assertStringContainsString('confirmée', $email['subject']);

    // Vérifier le destinataire
    $this->assertEquals($this->user->getEmail(), $email['to']);
  }
}
```

---

## Preview Symfony Mailer

```
/admin/config/system/mailer/policy

→ Cliquer sur une policy (ex: "User — Welcome")
→ "Preview" → visualiser l'email HTML dans le navigateur
→ "Send test" → envoyer à une adresse de test
```

```bash
# Previewer depuis Drush
drush php:eval "
\$email = \Drupal::service('email_factory')->newTypedEmail('mon_module', 'commande_confirmee');
echo \$email->getHtmlBody();  // Affiche le HTML de l'email
"
```

---

## Debug des Emails

```bash
# Vérifier que Symfony Mailer est actif et configuré
drush php:eval "
\$mailer_config = \Drupal::config('symfony_mailer.mailer_transport.default');
echo 'Transport: ' . \$mailer_config->get('plugin') . PHP_EOL;
echo 'DSN: ' . \$mailer_config->get('configuration.dsn') . PHP_EOL;
"

# Logs d'envoi
drush watchdog:show --type=symfony_mailer --count=20

# Test d'envoi simple
drush php:eval "
try {
  \Drupal::service('email_factory')
    ->newEmail()
    ->setTo(getenv('ADMIN_MAIL') ?: 'test@example.com')
    ->setSubject('Test — ' . date('Y-m-d H:i:s'))
    ->setHtmlBody('<p>Test email</p>')
    ->send();
  echo 'OK';
} catch (\Exception \$e) {
  echo 'ERREUR: ' . \$e->getMessage();
}
"

# Vérifier que Maildev reçoit bien
curl http://localhost:83/email | python3 -m json.tool | head -30
```
