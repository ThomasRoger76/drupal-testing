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
         printerClass="\Drupal\Tests\Listeners\HtmlOutputPrinter">

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

  <coverage>
    <include>
      <directory>web/modules/custom</directory>
    </include>
    <exclude>
      <directory>web/modules/custom/*/tests</directory>
    </exclude>
  </coverage>
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

### Configuration `.docker compose exec php/config.yaml` pour les tests

```yaml
# .docker compose exec php/config.yaml — extensions utiles pour les tests
webimage_extra_packages:
  - php8.1-xdebug  # Coverage
  
hooks:
  post-start:
    - exec: "mkdir -p /tmp/drupal-browsertest-output"
```

### Setup ChromeDriver (FunctionalJavascript)

```bash
# Ajouter le service Selenium dans docker-compose.yml
# module non nécessaire avec Docker Compose
docker compose restart php
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

## Annotations PHPUnit dans Drupal

```php
/**
 * Tests du service MonService.
 *
 * @group mon_module          ← Groupe pour filtrer avec --group
 * @group mon_module_unit     ← Sous-groupe optionnel
 * @coversDefaultClass \Drupal\mon_module\Service\MonService
 */
class MonServiceTest extends UnitTestCase {

  /**
   * @covers ::methodName
   * @dataProvider monDataProvider
   */
  public function testMethodName(): void {
    // ...
  }
}
```

En D10+ avec PHP 8.1+, les attributs PHP sont supportés par PHPUnit 10 :
```php
#[\PHPUnit\Framework\Attributes\Group('mon_module')]
#[\PHPUnit\Framework\Attributes\CoversClass(MonService::class)]
class MonServiceTest extends UnitTestCase {
  // ...
}
```

---

## Troubleshooting Infrastructure

| Erreur | Cause | Solution |
|--------|-------|---------|
| `Unable to find test modules` | Namespace PSR-4 incorrect | Vérifier `autoload-dev` dans `composer.json` |
| `SIMPLETEST_DB not set` | Variable d'env manquante | Définir dans `phpunit.xml` ou `.env` |
| `Could not connect to ChromeDriver` | Selenium non démarré | `# module non nécessaire avec Docker Compose |
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
| `Class not found` dans bootstrap | Bootstrap Drupal non chargé | Vérifier que `bootstrap="web/core/tests/bootstrap.php"` est dans phpunit.xml |
