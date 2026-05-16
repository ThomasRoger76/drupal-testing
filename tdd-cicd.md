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

## GitLab CI — Pipeline Complet Drupal 11

Pipeline en 3 stages : **validate** (phpcs + phpstan + rector) → **test** (unit/kernel/functional) → **coverage** (PCOV + seuil 70%).

```yaml
# .gitlab-ci.yml — Pipeline Drupal 11 complet
stages:
  - validate   # Analyse statique — secondes, bloque la suite si échec
  - test       # PHPUnit — Unit + Kernel (SQLite) + Functional (MariaDB)
  - coverage   # PCOV — couverture avec seuil minimum 70%

variables:
  COMPOSER_ALLOW_SUPERUSER: "1"
  COMPOSER_NO_INTERACTION: "1"
  # Image Drupal 11 / PHP 8.3
  PHP_IMAGE: "drupal:11-php8.3-apache"
  # Connexion MariaDB pour les Functional tests
  MYSQL_ROOT_PASSWORD: "root"
  MYSQL_DATABASE: "drupal_test"
  MYSQL_USER: "drupal"
  MYSQL_PASSWORD: "drupal"

# Template partagé — installation Composer + phpunit.xml
.base: &base
  image: "${PHP_IMAGE}"
  before_script:
    - composer install --no-progress --prefer-dist --optimize-autoloader
    - cp phpunit.xml.dist phpunit.xml

# ─────────────────────────────────────────────────────────
# STAGE 1 : VALIDATE
# ─────────────────────────────────────────────────────────

validate:phpcs:
  <<: *base
  stage: validate
  script:
    # Standard Drupal (2 espaces, PHPDoc obligatoire, etc.)
    - vendor/bin/phpcs \
        --standard=Drupal \
        --extensions=php,module,inc,install,theme \
        --report=checkstyle \
        --report-file=phpcs-report.xml \
        web/modules/custom \
        || (cat phpcs-report.xml && exit 1)
  artifacts:
    when: always
    reports:
      codequality: phpcs-report.xml
    expire_in: 1 week

validate:phpstan:
  <<: *base
  stage: validate
  script:
    # Niveau 6 avec règles Drupal — détecte dépréciations et erreurs de typage
    - vendor/bin/phpstan analyse \
        --configuration=phpstan.neon \
        --error-format=gitlab \
        --no-progress \
        > phpstan-report.json
  after_script:
    # Re-afficher en cas d'erreur pour debug
    - vendor/bin/phpstan analyse --configuration=phpstan.neon --no-progress || true
  artifacts:
    when: always
    reports:
      codequality: phpstan-report.json
    expire_in: 1 week
  allow_failure: false

validate:rector:
  <<: *base
  stage: validate
  script:
    # Dry-run — échoue si des règles D10→D11 / PHP 8.x ne sont pas appliquées
    - vendor/bin/rector process \
        --dry-run \
        --no-progress-bar \
        web/modules/custom
  allow_failure: false

# ─────────────────────────────────────────────────────────
# STAGE 2 : TEST
# ─────────────────────────────────────────────────────────

test:unit:
  <<: *base
  stage: test
  needs:
    - validate:phpcs
    - validate:phpstan
    - validate:rector
  script:
    # Unit tests — pas de DB, rapides
    - vendor/bin/phpunit \
        --testsuite unit \
        --log-junit phpunit-unit.xml \
        --colors=never \
        --no-progress
  artifacts:
    when: always
    reports:
      junit: phpunit-unit.xml
    expire_in: 1 week

test:kernel:
  <<: *base
  stage: test
  needs:
    - validate:phpcs
    - validate:phpstan
    - validate:rector
  variables:
    # SQLite en mémoire pour les Kernel tests — pas besoin de MariaDB
    SIMPLETEST_DB: "sqlite://localhost/:memory:"
  script:
    - vendor/bin/phpunit \
        --testsuite kernel \
        --log-junit phpunit-kernel.xml \
        --colors=never \
        --no-progress
  artifacts:
    when: always
    reports:
      junit: phpunit-kernel.xml
    expire_in: 1 week

test:functional:
  <<: *base
  stage: test
  needs:
    - test:unit
    - test:kernel
  services:
    # MariaDB pour l'installation Drupal complète (Functional tests)
    - name: mariadb:10.11
      alias: mariadb
  variables:
    SIMPLETEST_DB: "mysql://drupal:drupal@mariadb/drupal_test"
    SIMPLETEST_BASE_URL: "http://localhost"
    BROWSERTEST_OUTPUT_DIRECTORY: "/tmp/browser-output"
  before_script:
    - composer install --no-progress --prefer-dist --optimize-autoloader
    - cp phpunit.xml.dist phpunit.xml
    # Activer mod_rewrite (requis pour le routing Drupal)
    - a2enmod rewrite headers
    - service apache2 restart
    # Attendre que MariaDB soit prête (max 60s)
    - |
      for i in $(seq 1 30); do
        mysqladmin ping -h mariadb -u root -proot --silent && break
        echo "Attente MariaDB ($i/30)..."
        sleep 2
      done
    # Créer la DB de test
    - mysql -h mariadb -u root -proot -e \
        "CREATE DATABASE IF NOT EXISTS drupal_test CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;"
    - mysql -h mariadb -u root -proot -e \
        "GRANT ALL PRIVILEGES ON drupal_test.* TO 'drupal'@'%' IDENTIFIED BY 'drupal'; FLUSH PRIVILEGES;"
  script:
    - vendor/bin/phpunit \
        --testsuite functional \
        --log-junit phpunit-functional.xml \
        --colors=never \
        --no-progress
  artifacts:
    when: always
    reports:
      junit: phpunit-functional.xml
    paths:
      - /tmp/browser-output
    expire_in: 1 week

# ─────────────────────────────────────────────────────────
# STAGE 3 : COVERAGE — PCOV + seuil minimum 70%
# ─────────────────────────────────────────────────────────

coverage:
  <<: *base
  stage: coverage
  needs:
    - test:unit
    - test:kernel
  variables:
    # SQLite pour la couverture (unit + kernel uniquement)
    SIMPLETEST_DB: "sqlite://localhost/:memory:"
  before_script:
    - composer install --no-progress --prefer-dist --optimize-autoloader
    - cp phpunit.xml.dist phpunit.xml
    # Activer PCOV (plus rapide que Xdebug pour la couverture)
    # Les parenthèses groupent correctement les commandes de fallback
    - docker-php-ext-enable pcov 2>/dev/null || (pecl install pcov && docker-php-ext-enable pcov) || true
  script:
    # Couverture Unit + Kernel avec PCOV
    - php -d pcov.enabled=1 -d pcov.directory=web/modules/custom \
        vendor/bin/phpunit \
        --testsuite unit \
        --coverage-text \
        --coverage-clover coverage-unit.xml \
        --coverage-html coverage-unit-html/ \
        --colors=never \
        --no-progress
    - php -d pcov.enabled=1 -d pcov.directory=web/modules/custom \
        vendor/bin/phpunit \
        --testsuite kernel \
        --coverage-text \
        --coverage-clover coverage-kernel.xml \
        --colors=never \
        --no-progress
    # Vérifier le seuil minimum de couverture (70%)
    - |
      COVERAGE=$(php -r "
        \$xml = simplexml_load_file('coverage-unit.xml');
        \$metrics = \$xml->project->metrics;
        \$covered = (int)\$metrics['coveredstatements'];
        \$total   = (int)\$metrics['statements'];
        echo \$total > 0 ? round((\$covered / \$total) * 100, 1) : 0;
      ")
      echo "Couverture Unit Tests : ${COVERAGE}%"
      if (( $(echo "$COVERAGE < 70" | bc -l) )); then
        echo "ECHEC : couverture ${COVERAGE}% < seuil minimum 70%"
        exit 1
      fi
      echo "OK : couverture ${COVERAGE}% >= 70%"
  coverage: '/^\s*Lines:\s*(\d+\.\d+)%/'
  artifacts:
    when: always
    paths:
      - coverage-unit.xml
      - coverage-kernel.xml
      - coverage-unit-html/
    expire_in: 2 weeks
  allow_failure: false
```

### Variables GitLab CI à configurer dans Settings > CI/CD > Variables

| Variable | Valeur exemple | Description |
|----------|---------------|-------------|
| `MYSQL_ROOT_PASSWORD` | `root` | Mot de passe root MariaDB |
| `MYSQL_DATABASE` | `drupal_test` | Nom de la DB de test |
| `MYSQL_USER` | `drupal` | Utilisateur DB |
| `MYSQL_PASSWORD` | `drupal` | Mot de passe utilisateur DB |

---

## Code Coverage

```bash
# Nécessite Xdebug ou PCOV
docker compose exec php vendor/bin/phpunit --coverage-html coverage/ --group mon_module

# Coverage en texte (terminal)
docker compose exec php vendor/bin/phpunit --coverage-text --group mon_module

# Coverage XML (pour CI/SonarQube)
docker compose exec php vendor/bin/phpunit --coverage-clover coverage.xml --group mon_module

# Activer PCOV dans le container PHP (plus rapide que Xdebug)
docker compose exec php php -d pcov.enabled=1 vendor/bin/phpunit --coverage-html coverage/
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
	docker compose exec php vendor/bin/phpunit --testsuite unit --group $(MODULE)

test-kernel:
	docker compose exec php vendor/bin/phpunit --testsuite kernel --group $(MODULE)

test-functional:
	docker compose exec php vendor/bin/phpunit --testsuite functional --group $(MODULE)

test-all:
	docker compose exec php vendor/bin/phpunit --group $(MODULE)

test-coverage:
	docker compose exec php vendor/bin/phpunit --group $(MODULE) --coverage-html coverage/

# Usage: make test-unit MODULE=mon_module
```

### PHPStan pour la qualité (complément aux tests)

```bash
# Analyse statique avant les tests
docker compose exec php vendor/bin/phpstan analyse web/modules/custom/ --level=5

# Avec le niveau Drupal
docker compose exec php vendor/bin/phpstan analyse --configuration=phpstan.neon
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
