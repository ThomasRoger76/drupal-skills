# Infrastructure de Test

## Structure des Fichiers de Test dans un Module

```
mon_module/
└── tests/
    └── src/
        ├── Unit/                        # Tests unitaires (sans Drupal)
        │   └── MonServiceTest.php
        ├── Kernel/                      # Tests d'intégration légers
        │   └── MonKernelTest.php
        ├── Functional/                  # Tests bout-en-bout (navigateur simulé)
        │   └── MonFunctionalTest.php
        └── FunctionalJavascript/        # Tests avec JS réel
            └── MonJsTest.php
```

**Namespace convention :**
- Unit → `Drupal\Tests\mon_module\Unit\`
- Kernel → `Drupal\Tests\mon_module\Kernel\`
- Functional → `Drupal\Tests\mon_module\Functional\`
- FunctionalJavascript → `Drupal\Tests\mon_module\FunctionalJavascript\`

---

## `phpunit.xml` — Configuration Complète

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
         bootstrap="web/core/tests/bootstrap.php"
         colors="true"
         beStrictAboutChangesToGlobalState="true"
         cacheDirectory=".phpunit.cache"
         failOnWarning="true">

  <!--
    PHPUnit 10/11 : `printerClass` a été supprimé. Le rendu HTML des résultats
    BrowserTest passe désormais par l'extension Drupal, enregistrée ainsi :
  -->
  <extensions>
    <bootstrap class="Drupal\TestTools\Extension\HtmlLogging\HtmlOutputLogger">
      <parameter name="outputDirectory" value="/tmp/drupal-browsertest-output"/>
    </bootstrap>
  </extensions>

  <php>
    <!-- URL de base du site (pour les tests Functional) -->
    <env name="SIMPLETEST_BASE_URL" value="http://localhost"/>

    <!-- Base de données de test -->
    <env name="SIMPLETEST_DB" value="mysql://drupal:drupal@db/drupal_test"/>
    <!-- Ou SQLite pour les Kernel tests en local : -->
    <!-- <env name="SIMPLETEST_DB" value="sqlite://localhost/sites/default/files/.ht.sqlite"/> -->

    <!-- Répertoire pour les screenshots des tests Functional -->
    <env name="BROWSERTEST_OUTPUT_DIRECTORY" value="/tmp/drupal-browsertest-output"/>
    <env name="BROWSERTEST_OUTPUT_BASE_URL" value="http://localhost"/>

    <!-- ChromeDriver pour les FunctionalJavascript tests -->
    <env name="MINK_DRIVER_ARGS_WEBDRIVER" value='["chrome", {"browserName":"chrome","goog:chromeOptions":{"args":["--disable-gpu","--headless","--no-sandbox","--disable-dev-shm-usage"]}}, "http://selenium:4444/wd/hub"]'/>
  </php>

  <testsuites>
    <testsuite name="unit">
      <directory>web/modules/custom/*/tests/src/Unit</directory>
      <directory>web/modules/contrib/*/tests/src/Unit</directory>
    </testsuite>
    <testsuite name="kernel">
      <directory>web/modules/custom/*/tests/src/Kernel</directory>
    </testsuite>
    <testsuite name="functional">
      <directory>web/modules/custom/*/tests/src/Functional</directory>
    </testsuite>
    <testsuite name="javascript">
      <directory>web/modules/custom/*/tests/src/FunctionalJavascript</directory>
    </testsuite>
  </testsuites>

  <!--
    PHPUnit 10/11 : la définition des fichiers couverts se fait dans <source>,
    plus dans <coverage><include> (syntaxe PHPUnit 9, supprimée).
  -->
  <source>
    <include>
      <directory>web/modules/custom</directory>
    </include>
    <exclude>
      <directory>web/modules/custom/*/tests</directory>
    </exclude>
  </source>
</phpunit>
```

---

## Lancer les Tests avec Docker Compose

```bash
# Lancer TOUS les Unit tests
docker compose exec php vendor/bin/phpunit --testsuite unit

# Lancer les tests d'un module spécifique (par @group)
docker compose exec php vendor/bin/phpunit --group mon_module

# Lancer un fichier de test spécifique
docker compose exec php vendor/bin/phpunit web/modules/custom/mon_module/tests/src/Unit/MonServiceTest.php

# Lancer une méthode de test spécifique
docker compose exec php vendor/bin/phpunit --filter testMethodName web/modules/custom/mon_module/tests/src/Unit/MonServiceTest.php

# Lancer Kernel tests
docker compose exec php vendor/bin/phpunit --testsuite kernel --group mon_module

# Lancer Functional tests
docker compose exec php vendor/bin/phpunit --testsuite functional --group mon_module

# Avec verbose output
docker compose exec php vendor/bin/phpunit --verbose --group mon_module

# Avec code coverage (nécessite Xdebug ou PCOV)
docker compose exec php vendor/bin/phpunit --coverage-html coverage/ --group mon_module

# Via Drush (D9+)
docker compose exec php drush test:run --types=PHPUnit-Unit mon_module
docker compose exec php drush test:run --types=PHPUnit-Kernel mon_module
```

---

## Setup Docker Compose pour les Tests Fonctionnels

### Base de données de test

```bash
# Créer une DB de test dédiée (recommandé pour éviter de polluer la DB principale)
docker compose exec php mysql -e "CREATE DATABASE IF NOT EXISTS drupal_test;"
docker compose exec php mysql -e "GRANT ALL ON drupal_test.* TO 'drupal'@'%';"
```

### Préparation du dossier de sortie BrowserTest

```yaml
# docker-compose.yml — créer le dossier des screenshots au démarrage du container PHP
services:
  php:
    # ... configuration existante (build, volumes, etc.)
    # Xdebug pour la couverture est généralement déjà dans l'image PHP custom
    command: >
      sh -c "mkdir -p /tmp/drupal-browsertest-output && php-fpm"
```

### Setup ChromeDriver (FunctionalJavascript)

```bash
# Ajouter un service Selenium dans docker-compose.yml puis le démarrer
docker compose up -d selenium
```

```yaml
# docker-compose.yml — service Selenium Standalone Chrome
services:
  selenium:
    image: selenium/standalone-chrome:latest
    shm_size: '2gb'
    ports:
      - "4444:4444"
```

```xml
<!-- phpunit.xml — configuration ChromeDriver Docker Compose -->
<env name="MINK_DRIVER_ARGS_WEBDRIVER" value='["chrome", {"browserName":"chrome","goog:chromeOptions":{"args":["--disable-gpu","--headless","--no-sandbox"]}}, "http://selenium:4444/wd/hub"]'/>
```

---

## Variables d'Environnement — Référence

| Variable | Description | Exemple |
|----------|-------------|---------|
| `SIMPLETEST_BASE_URL` | URL du site Drupal | `http://localhost` |
| `SIMPLETEST_DB` | DSN de la DB de test | `mysql://drupal:drupal@db/drupal_test` |
| `BROWSERTEST_OUTPUT_DIRECTORY` | Dossier screenshots | `/tmp/drupal-test-output` |
| `BROWSERTEST_OUTPUT_BASE_URL` | URL pour les liens de screenshots | `http://localhost` |
| `MINK_DRIVER_ARGS_WEBDRIVER` | Config ChromeDriver (JSON) | Voir phpunit.xml ci-dessus |
| `DTT_BASE_URL` | URL pour Drupal Test Traits | `http://localhost` |
| `XDEBUG_MODE` | Mode Xdebug pour coverage | `coverage` |

---

## Métadonnées PHPUnit — Attributs PHP (standard D11) vs Annotations

**Standard recommandé (D10.1+, obligatoire en D11 / PHPUnit 11) — attributs PHP :**

```php
use PHPUnit\Framework\Attributes\CoversClass;
use PHPUnit\Framework\Attributes\CoversMethod;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\Attributes\Group;

#[Group('mon_module')]
#[CoversClass(MonService::class)]
final class MonServiceTest extends UnitTestCase {

  #[CoversMethod('methodName')]
  #[DataProvider('monDataProvider')]
  public function testMethodName(): void {
    // ...
  }
}
```

**Forme historique (annotations docblock) — D8/D9/D10, dépréciée et supprimée dans PHPUnit 11 :**

```php
/**
 * @group mon_module
 * @coversDefaultClass \Drupal\mon_module\Service\MonService
 */
class MonServiceTest extends UnitTestCase {

  /**
   * @covers ::methodName
   * @dataProvider monDataProvider
   */
  public function testMethodName(): void {}
}
```

> Sur un projet D11, n'utiliser que les attributs PHP. Rector (`palantirnet/drupal-rector`)
> convertit automatiquement les annotations en attributs — voir [static-analysis.md](static-analysis.md).

---

## Troubleshooting Infrastructure

| Erreur | Cause | Solution |
|--------|-------|---------|
| `Unable to find test modules` | Namespace PSR-4 incorrect | Vérifier `autoload-dev` dans `composer.json` |
| `SIMPLETEST_DB not set` | Variable d'env manquante | Définir dans `phpunit.xml` ou `.env` |
| `Could not connect to ChromeDriver` | Selenium non démarré | `docker compose up -d selenium` puis vérifier `docker compose ps` |
| Tests kernel plus lents que prévu | SQLite non configuré | Utiliser `sqlite://` dans SIMPLETEST_DB pour les kernel tests |
| `Class not found` dans bootstrap | Bootstrap Drupal non chargé | Vérifier que `bootstrap="web/core/tests/bootstrap.php"` est dans phpunit.xml |

---

## Behat — Tests BDD Comportementaux

Behat permet d'écrire des tests en langage naturel (Gherkin : Given/When/Then). Idéal pour les projets avec exigences fonctionnelles formalisées côté client/QA.

### Installation

```bash
composer require --dev behat/behat behat/mink behat/mink-extension drupal/drupal-extension
```

### Configuration `behat.yml`

```yaml
default:
  suites:
    default:
      contexts:
        - Drupal\DrupalExtension\Context\DrupalContext
        - Drupal\DrupalExtension\Context\MinkContext
        - Drupal\DrupalExtension\Context\MessageContext
  extensions:
    Behat\MinkExtension:
      goutte: ~
      selenium2:
        wd_host: "http://selenium:4444/wd/hub"
      base_url: http://localhost
    Drupal\DrupalExtension:
      blackbox: ~
      api_driver: drush
      drush:
        root: /var/www/html/web
```

### Exemple de scénario Gherkin

```gherkin
# features/inscription.feature
Feature: Inscription utilisateur
  En tant que visiteur
  Je veux pouvoir m'inscrire
  Afin d'accéder à l'espace membre

  Scenario: Inscription valide
    Given je suis sur "/user/register"
    When je remplis "Email" avec "test@example.com"
    And je remplis "Nom d'utilisateur" avec "testuser"
    And je presse "Créer un compte"
    Then je dois voir "Un email de confirmation a été envoyé"

  Scenario: Accès refusé sans connexion
    Given je ne suis pas connecté
    When je vais sur "/admin"
    Then je dois être sur "/user/login"
    And le code de réponse HTTP doit être 403
```

### Lancer les tests Behat

```bash
# Tous les scénarios
vendor/bin/behat

# Un fichier spécifique
vendor/bin/behat features/inscription.feature

# Un scénario par ligne
vendor/bin/behat features/inscription.feature:12

# Lister les étapes disponibles
vendor/bin/behat -dl

# Avec Docker Compose
docker compose exec php vendor/bin/behat
```

### Quand choisir Behat vs PHPUnit Functional ?

| Critère | Behat | PHPUnit Functional |
|---------|-------|-------------------|
| Public cible | PO, QA, testeurs non-tech | Développeurs |
| Syntaxe | Gherkin (texte naturel) | PHP |
| Vitesse | Lent (navigateur réel) | Moyen (navigateur simulé) |
| JS support | ✅ Selenium | ✅ WebDriverTestBase |
| CI/CD | ✅ (config behat.yml) | ✅ (phpunit.xml) |
| Idéal pour | Scénarios de recette client | Tests d'intégration dev |
