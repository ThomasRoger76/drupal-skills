# Changelog — drupal-testing

---

## v1.2 — 2026-06-09

**Audit qualité — correction de défauts réels (cohérence Docker natif + PHPUnit 10/11) :**

`infrastructure.md`
- Artefacts de remplacement automatique DDEV→Docker corrigés : `.docker compose exec php/config.yaml` (nom de fichier corrompu) remplacé par un vrai exemple `docker-compose.yml` ; commentaires fantômes « # module non nécessaire » et `docker compose restart php` remplacés par un service `selenium/standalone-chrome` + `docker compose up -d selenium`
- Solution tronquée du tableau troubleshooting ChromeDriver complétée
- Ligne dupliquée parasite (bootstrap) retirée du tableau Behat vs PHPUnit
- `phpunit.xml` : `printerClass` (supprimé en PHPUnit 10) → extension `HtmlOutputLogger` ; ajout `cacheDirectory` / `failOnWarning` ; bloc couverture migré de `<coverage><include>` vers `<source>` (syntaxe PHPUnit 10/11)
- Section « Annotations PHPUnit » réécrite : attributs PHP présentés comme standard D11, annotations docblock comme forme historique dépréciée

`javascript-tests.md`
- Mêmes artefacts Selenium corrigés (bloc d'install + troubleshooting `docker compose up -d selenium`)

`static-analysis.md`
- Pipeline GitLab CI : suppression des `docker compose exec php …` à l'intérieur de jobs CI (le job tourne déjà dans le container) pour phpcs et infection — appel direct de `vendor/bin`

`tdd-cicd.md`
- `phpunit.xml` couverture : `<include>/<exclude>` déplacés de `<coverage>` vers `<source>` (PHPUnit 10/11)

`drupal-test-traits.md`
- `phpunit.xml` : `verbose="true"` (supprimé en PHPUnit 10) → `displayDetailsOnTestsThatTriggerWarnings` + `cacheDirectory`

`unit-tests.md`
- `@covers ::calculer` (docblock) → `#[CoversMethod('calculer')]` pour cohérence avec le reste du fichier (attributs PHP)

`SKILL.md`
- Anti-pattern coverage : `forceCoversAnnotation` (supprimé en PHPUnit 10) remplacé par `--min-coverage` / `requireCoverageMetadata`

`lessons.md`
- 3 leçons ajoutées : schéma `phpunit.xml` PHPUnit 9 vs 10/11, `docker compose exec` interdit en CI, Selenium comme service Docker Compose

---

## v1.1 — 2026-05-16

**Bug bash corrigé :**
- `tdd-cicd.md` stage coverage : `docker-php-ext-enable pcov || pecl install pcov && docker-php-ext-enable pcov || true` → priorité d'opérateurs bash incorrecte en cas de chaîne `||...&&`. Corrigé en `docker-php-ext-enable pcov 2>/dev/null || (pecl install pcov && docker-php-ext-enable pcov) || true` (parenthèses pour grouper correctement le fallback)

**lessons.md :**
- Date `2026-05-14` ajoutée sur la dernière leçon "Module d'import/synchro API" — cohérence de format avec les autres entrées

---

## v1.0 — 2026-05-14

**Création initiale**

### Couverture

**`SKILL.md`**
- Pyramide de décision (Unit → Kernel → Functional → FunctionalJavascript)
- Quick Decision Table (16 entrées)
- Anti-patterns critiques (8 entrées — sleep(), defaultTheme, mocks, etc.)
- Table versioning D8→D11 (PHPUnit 6→10, PHP Attributes, drush test:run)
- Section Auto-Amélioration

**`infrastructure.md`**
- Structure dossiers tests dans un module (Unit/Kernel/Functional/FunctionalJavascript)
- `phpunit.xml` complet (SIMPLETEST_BASE_URL, SIMPLETEST_DB, BROWSERTEST_OUTPUT_DIRECTORY, MINK_DRIVER_ARGS_WEBDRIVER)
- Toutes les commandes `vendor/bin/phpunit` (--testsuite, --group, --filter, --coverage-*)
- Setup DDEV : DB de test, config, ChromeDriver via docker compose exec php-selenium-standalone-chrome
- Variables d'environnement référence
- Annotations PHPUnit (@group, @covers) + PHP Attributes (D10+)
- Tableau troubleshooting infrastructure

**`unit-tests.md`**
- Structure `UnitTestCase` complète
- Mocking : `createMock()`, `willReturn()`, `willReturnMap()`, `willReturnCallback()`, `expects($this->exactly(N))`, `with()`
- DataProviders avec PHPUnit 10 Attributes
- Test des exceptions (`expectException`, `expectExceptionMessage`)
- `StringTranslationTrait` stub pour `$this->t()`
- Assertions complètes (assertSame, assertEquals, assertNull, assertCount, assertStringContainsString, etc.)
- Pattern pour services utilisant `\Drupal::` en interne

**`kernel-tests.md`**
- Structure `KernelTestBase` avec `$modules`
- Référence complète `installEntitySchema()`, `installSchema()`, `installConfig()`
- Tests Entity API (CRUD complet : create, load, modify, delete)
- Tests Config API (défaults, modification)
- Tests de services via container
- Tests de hooks Drupal (`hook_entity_presave`, `hook_node_access`)
- Tests EntityQuery avec conditions
- Tests de formulaires (buildForm, submitForm via FormState)
- Modules courants à inclure + règle du minimum
- Création d'utilisateurs avec rôles dans Kernel tests

**`functional-tests.md`**
- Structure `BrowserTestBase` + `$defaultTheme` obligatoire
- Navigation et assertions HTTP (200, 403, 404, redirections)
- Gestion utilisateurs : `drupalCreateUser()`, `drupalLogin()`, `drupalLogout()`, super admin
- Soumission de formulaires : `submitForm()`, validation, création de nœuds
- Assertions complètes (statusCodeEquals, pageTextContains, elementExists, fieldValueEquals, addressEquals, responseHeaderEquals, etc.)
- Test d'endpoints REST/JSON (json_decode, Content-Type)
- Helpers : `clickLink()`, `getSession()`, `drupalCreateNode()`, `drupalCreateUser()` avec DataProvider

**`javascript-tests.md`**
- Prérequis ChromeDriver avec DDEV (docker compose exec php-selenium-standalone-chrome)
- Structure `WebDriverTestBase`
- Méthodes d'attente AJAX : `waitForElement()`, `waitForText()`, `waitForElementRemoved()`, `assertWaitOnAjaxRequest()`
- Exemple avec vs sans sleep (bon vs mauvais pattern)
- Interactions : clic, autocomplete, modales, drag-and-drop
- Exécution JavaScript : `evaluateScript()`, `executeScript()`
- Screenshots : `createScreenshot()`, `onNotSuccessfulTest()` automatique
- Tableau comparatif BrowserTestBase vs WebDriverTestBase
- Troubleshooting FunctionalJavascript

**`tdd-cicd.md`**
- TDD cycle RED → GREEN → REFACTOR avec exemple Drupal complet
- Règles TDD pour Drupal (ne pas tester le core, nommage, un test à la fois)
- GitHub Actions complet : Unit+Kernel job + Functional job séparés, MySQL service, coverage Codecov
- GitLab CI complet : template, 3 stages, artifacts JUnit, manual pour functional
- Code coverage : `--coverage-html`, `--coverage-clover`, PCOV dans DDEV
- Configuration coverage dans `phpunit.xml` (include/exclude, report HTML + Clover)
- Stratégie pipeline (lint → unit → kernel → functional → js, du plus rapide au plus lent)
- Makefile pour simplifier les commandes locales
- PHPStan intégration avec plugin drupal

**`lessons.md`**
- 9 leçons pré-remplies avec symptôme/cause/correct/prévention :
  - `$defaultTheme` manquant
  - `$modules` incomplet
  - `installEntitySchema()` oublié
  - `installSchema('system', ['sequences'])` oublié
  - `sleep()` vs `waitFor*()`
  - `pageTextContains()` et son périmètre réel
  - Kernel tests trop lents (trop de modules)
  - Conflits de mocks entre setUp() et tests
  - `assertWaitOnAjaxRequest()` timeout CI
  - `Node::create()` sans `uid`

---

## Compatibilité Drupal

| Skill version | Drupal | PHPUnit | PHP minimum |
|--------------|--------|---------|-------------|
| v1.0 | D9, D10, D11 | 9.x (D9), 10.x (D10+) | 8.1 (D10) / 8.3 (D11) |
