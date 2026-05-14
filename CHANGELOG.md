# Changelog — drupal-testing

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
- Setup DDEV : DB de test, config, ChromeDriver via ddev-selenium-standalone-chrome
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
- Prérequis ChromeDriver avec DDEV (ddev-selenium-standalone-chrome)
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
