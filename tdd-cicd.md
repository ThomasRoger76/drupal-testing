# TDD & CI/CD

## Le Cycle TDD — RED → GREEN → REFACTOR

```
    ┌─────────────────────────────────────────┐
    │  1. RED — Écrire un test qui ÉCHOUE     │
    │     (le code n'existe pas encore)       │
    └──────────────────┬──────────────────────┘
                       ↓
    ┌─────────────────────────────────────────┐
    │  2. GREEN — Écrire le code MINIMUM      │
    │     qui fait passer le test             │
    └──────────────────┬──────────────────────┘
                       ↓
    ┌─────────────────────────────────────────┐
    │  3. REFACTOR — Améliorer le code        │
    │     sans casser les tests               │
    └──────────────────┬──────────────────────┘
                       ↓ (recommencer)
```

### Exemple TDD concret pour Drupal

```php
// ÉTAPE 1 — RED : le test échoue car PrixCalculator n'existe pas

// tests/src/Unit/PrixCalculatorTest.php
public function testCalculerAvecTaxe(): void {
  $calc = new PrixCalculator();
  $this->assertEquals(120.0, $calc->calculer(100.0, 0.20));
}
// → PHPUnit Error: Class PrixCalculator not found

// ÉTAPE 2 — GREEN : code minimal pour faire passer
class PrixCalculator {
  public function calculer(float $base, float $taux): float {
    return $base * (1 + $taux);  // Le minimum qui passe
  }
}
// → OK : 1 test passed

// ÉTAPE 3 — REFACTOR : ajouter validation, typage, etc.
class PrixCalculator {
  public function calculer(float $base, float $taux): float {
    if ($base < 0) {
      throw new \InvalidArgumentException('Le prix ne peut pas être négatif');
    }
    if ($taux < 0 || $taux > 1) {
      throw new \RangeException('Le taux doit être entre 0 et 1');
    }
    return $base + ($base * $taux);
  }
}
// → Ajouter les tests pour les nouvelles validations
```

### Règles TDD pour Drupal

- Écrire **un seul test à la fois** → fail → code minimal → pass → suivant
- Ne pas tester les APIs Drupal core (elles ont leurs propres tests)
- Nommer les tests `test + comportement attendu` : `testPrixNegatifLeveException`
- Chaque test doit échouer **pour la bonne raison** — lire le message d'erreur
- Si le refactoring casse un test → le comportement a changé → décision consciente

---

## GitHub Actions — Intégration Continue

```yaml
# .github/workflows/drupal-tests.yml
name: Drupal PHPUnit Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  unit-kernel:
    name: "Unit & Kernel Tests"
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: drupal_test
          MYSQL_USER: drupal
          MYSQL_PASSWORD: drupal
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: gd, pdo_mysql, xml
          coverage: pcov

      - name: Install Composer dependencies
        run: composer install --no-progress --prefer-dist --optimize-autoloader

      - name: Setup phpunit.xml
        run: |
          cp phpunit.xml.dist phpunit.xml
          sed -i 's|SIMPLETEST_DB.*|SIMPLETEST_DB" value="mysql://drupal:drupal@127.0.0.1/drupal_test"/>|' phpunit.xml

      - name: Run Unit Tests
        run: vendor/bin/phpunit --testsuite unit --coverage-clover coverage-unit.xml

      - name: Run Kernel Tests
        run: vendor/bin/phpunit --testsuite kernel
        env:
          SIMPLETEST_DB: "mysql://drupal:drupal@127.0.0.1/drupal_test"

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: coverage-unit.xml

  functional:
    name: "Functional Tests"
    runs-on: ubuntu-latest
    needs: unit-kernel  # Ne lance qu'après Unit/Kernel OK

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: drupal_test
        ports:
          - 3306:3306

    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: gd, pdo_mysql, xml

      - run: composer install --no-progress --prefer-dist

      # ⚠️ php -S ne supporte PAS mod_rewrite/.htaccess → Drupal routing cassé
      # Utiliser Apache avec mod_rewrite activé
      - name: Configure Apache for Drupal
        run: |
          sudo apt-get install -y apache2
          sudo a2enmod rewrite headers
          sudo ln -sf $GITHUB_WORKSPACE/web /var/www/html/drupal
          echo '<VirtualHost *:8080>
            DocumentRoot /var/www/html/drupal
            <Directory /var/www/html/drupal>
              AllowOverride All
              Require all granted
            </Directory>
          </VirtualHost>' | sudo tee /etc/apache2/sites-available/drupal.conf
          sudo a2ensite drupal
          sudo sed -i 's/Listen 80/Listen 8080/' /etc/apache2/ports.conf
          sudo service apache2 restart

      - name: Run Functional Tests
        run: vendor/bin/phpunit --testsuite functional
        env:
          SIMPLETEST_DB: "mysql://drupal:drupal@127.0.0.1/drupal_test"
          SIMPLETEST_BASE_URL: "http://localhost:8080"
          BROWSERTEST_OUTPUT_DIRECTORY: "/tmp/browser-output"
```

---

## GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - test

variables:
  MYSQL_ROOT_PASSWORD: root
  MYSQL_DATABASE: drupal_test
  MYSQL_USER: drupal
  MYSQL_PASSWORD: drupal
  SIMPLETEST_DB: "mysql://drupal:drupal@mysql/drupal_test"

.test_template: &test_template
  image: drupal:10-php8.3-apache
  services:
    - mysql:8.0
  before_script:
    - composer install --no-progress --prefer-dist
    - cp phpunit.xml.dist phpunit.xml

test:unit:
  <<: *test_template
  stage: test
  script:
    - vendor/bin/phpunit --testsuite unit --colors=never
  coverage: '/^\s*Lines:\s*\d+.\d+\%/'
  artifacts:
    reports:
      junit: phpunit-unit.xml
    when: always

test:kernel:
  <<: *test_template
  stage: test
  script:
    - vendor/bin/phpunit --testsuite kernel --log-junit phpunit-kernel.xml
  artifacts:
    reports:
      junit: phpunit-kernel.xml

test:functional:
  <<: *test_template
  stage: test
  script:
    - vendor/bin/phpunit --testsuite functional
  when: manual  # Lancer manuellement (plus lent)
  allow_failure: false
```

---

## Code Coverage

```bash
# Nécessite Xdebug ou PCOV
ddev exec vendor/bin/phpunit --coverage-html coverage/ --group mon_module

# Coverage en texte (terminal)
ddev exec vendor/bin/phpunit --coverage-text --group mon_module

# Coverage XML (pour CI/SonarQube)
ddev exec vendor/bin/phpunit --coverage-clover coverage.xml --group mon_module

# Activer PCOV dans DDEV (plus rapide que Xdebug)
ddev exec php -d pcov.enabled=1 vendor/bin/phpunit --coverage-html coverage/
```

### Configurer le rapport dans `phpunit.xml`

```xml
<coverage>
  <include>
    <directory>web/modules/custom/mon_module/src</directory>
  </include>
  <exclude>
    <directory>web/modules/custom/mon_module/tests</directory>
    <!-- Exclure les fichiers de "plumbing" non testables -->
    <file>web/modules/custom/mon_module/mon_module.module</file>
  </exclude>
  <report>
    <html outputDirectory="coverage/html"/>
    <clover outputFile="coverage/clover.xml"/>
    <text outputFile="php://stdout" showOnlySummary="true"/>
  </report>
</coverage>
```

---

## Bonnes Pratiques CI/CD pour Drupal

### Stratégie de tests par rapidité

```yaml
# Pipeline optimisé : du plus rapide au plus lent
stages:
  - lint       # PHPCS, PHPStan — secondes
  - unit       # Unit tests — quelques secondes
  - kernel     # Kernel tests — 30s-2min
  - functional # Functional tests — 2-10min
  - js         # FunctionalJavascript — 5-20min (optionnel en MR)
```

### `Makefile` pour simplifier

```makefile
# Makefile à la racine du projet
test-unit:
	ddev exec vendor/bin/phpunit --testsuite unit --group $(MODULE)

test-kernel:
	ddev exec vendor/bin/phpunit --testsuite kernel --group $(MODULE)

test-functional:
	ddev exec vendor/bin/phpunit --testsuite functional --group $(MODULE)

test-all:
	ddev exec vendor/bin/phpunit --group $(MODULE)

test-coverage:
	ddev exec vendor/bin/phpunit --group $(MODULE) --coverage-html coverage/

# Usage: make test-unit MODULE=mon_module
```

### PHPStan pour la qualité (complément aux tests)

```bash
# Analyse statique avant les tests
ddev exec vendor/bin/phpstan analyse web/modules/custom/ --level=5

# Avec le niveau Drupal
ddev exec vendor/bin/phpstan analyse --configuration=phpstan.neon
```

```neon
# phpstan.neon
includes:
  - vendor/mglaman/phpstan-drupal/extension.neon

parameters:
  level: 5
  paths:
    - web/modules/custom/mon_module/src
```
