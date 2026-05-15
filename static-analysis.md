# Analyse Statique — PHPStan, PHPCS, Rector

## Vue d'ensemble

L'analyse statique détecte les bugs et les violations de style **sans exécuter le code**. Obligatoire avant chaque commit dans un pipeline CI sérieux.

```
PHPStan    → bugs de typage, méthodes inexistantes, dépréciations
PHPCS      → style de code (standard Drupal : 2 espaces, pas de PSR-12)
PHPCBF     → correction automatique des violations PHPCS
drupal-check → scan dépréciations avant upgrade
Rector     → refactoring automatique (D9→D10→D11, PHP 8.x)
```

---

## PHPStan avec règles Drupal

### Installation

```bash
composer require --dev \
  phpstan/phpstan \
  mglaman/phpstan-drupal \
  phpstan/phpstan-deprecation-rules \
  jangregor/phpstan-prophecy
```

### `phpstan.neon` — configuration complète

```neon
# phpstan.neon (à la racine du projet)
includes:
  - vendor/mglaman/phpstan-drupal/extension.neon
  - vendor/phpstan/phpstan-deprecation-rules/rules.neon

parameters:
  level: 6

  paths:
    - web/modules/custom

  # Exclure les dossiers non pertinents
  excludePaths:
    - web/modules/custom/*/tests
    - web/modules/custom/*/node_modules

  # Ignorer les faux-positifs Drupal connus
  ignoreErrors:
    # Services dynamiques (\Drupal::service() retourne mixed)
    - '#Call to an undefined method Drupal\\Core\\.*#'
    # Magic methods sur les entités Drupal (get/set champs)
    - '#Call to an undefined method Drupal\\Core\\Entity\\.*::get\(\)#'

  # Chemin vers les stubs Drupal (fournis par mglaman/phpstan-drupal)
  drupal:
    drupal_root: web
```

### Niveaux PHPStan expliqués

| Niveau | Ce qui est vérifié | Recommandation |
|--------|-------------------|----------------|
| 0 | Erreurs basiques (classes/fonctions inconnues) | Minimum absolu |
| 1 | Variables potentiellement `null`, types basiques | CI minimal |
| 2 | Types retour incertains | Nouveau projet |
| 3 | Paramètres de méthode | Recommandé |
| 4 | Types scalaires imprécis | Bonne pratique |
| 5 | Méthodes appelées sur `mixed` | Qualité pro |
| **6** | **Typage strict des tableaux** | **Standard Drupal** |
| 7 | Propriétés de classe non initialisées | Strict |
| 8 | Propriétés `nullable` incorrectes | Très strict |
| 9 | Analyse complète (max) | Projets critiques |

### Lancer PHPStan

```bash
# Analyse complète (niveau 6)
docker compose exec php vendor/bin/phpstan analyse \
  web/modules/custom \
  --configuration=phpstan.neon \
  --no-progress

# Analyse d'un seul module
docker compose exec php vendor/bin/phpstan analyse \
  web/modules/custom/mon_module/src \
  --level=6 \
  --no-progress

# Générer une baseline (ignorer les erreurs existantes)
docker compose exec php vendor/bin/phpstan analyse \
  --generate-baseline phpstan-baseline.neon \
  --configuration=phpstan.neon

# Après génération de la baseline, l'inclure dans phpstan.neon :
# includes:
#   - phpstan-baseline.neon
```

### Exemples d'erreurs Drupal spécifiques

```
# Erreur : utiliser un service déprécié (phpstan-deprecation-rules)
------ -------------------------------------------------------
 Line  web/modules/custom/mon_module/src/Service/ArticleService.php
------ -------------------------------------------------------
  42   Call to deprecated method getStorage() of class Drupal\Core\Entity\EntityManager.
       Use Drupal\Core\Entity\EntityTypeManager instead.
------ -------------------------------------------------------

# Erreur : type de retour incorrect
------ -------------------------------------------------------
 Line  web/modules/custom/mon_module/src/Controller/ArticleController.php
------ -------------------------------------------------------
  28   Method Drupal\mon_module\Controller\ArticleController::list()
       should return Symfony\Component\HttpFoundation\Response
       but returns array.
------ -------------------------------------------------------

# Erreur : paramètre nullable non géré
------ -------------------------------------------------------
 Line  web/modules/custom/mon_module/src/Service/PrixCalculator.php
------ -------------------------------------------------------
  15   Cannot call method getValue() on Drupal\Core\Field\FieldItemListInterface|null.
------ -------------------------------------------------------
```

### Corriger une erreur PHPStan typique

```php
// ❌ PHPStan niveau 5+ : Cannot call method on possibly null
$node = Node::load($nid);
$title = $node->label();  // $node peut être null !

// ✅ Corriger
$node = Node::load($nid);
if ($node === NULL) {
  return NULL;
}
$title = $node->label();

// ❌ Type de retour manquant
public function getArticles() {
  return $this->storage->loadMultiple();
}

// ✅ Type explicite
/** @return \Drupal\node\NodeInterface[] */
public function getArticles(): array {
  return $this->storage->loadMultiple();
}
```

---

## PHP CodeSniffer — Standard Drupal

### Installation

```bash
composer require --dev \
  squizlabs/php_codesniffer \
  drupal/coder
```

### `phpcs.xml` — configuration complète

```xml
<?xml version="1.0"?>
<ruleset name="Projet Drupal">
  <description>Coding standards Drupal pour ce projet.</description>

  <!-- Dossiers à analyser -->
  <file>web/modules/custom</file>
  <file>web/themes/custom</file>

  <!-- Exclure les dossiers non pertinents -->
  <exclude-pattern>*/tests/src/*</exclude-pattern>
  <exclude-pattern>*/node_modules/*</exclude-pattern>
  <exclude-pattern>*/vendor/*</exclude-pattern>

  <!-- Standard Drupal (2 espaces, conventions Drupal) -->
  <rule ref="Drupal"/>
  <rule ref="DrupalPractice"/>

  <!-- Extensions à analyser -->
  <arg name="extensions" value="php,module,inc,install,test,profile,theme,css,info,txt,md,yml"/>

  <!-- Affichage coloré + progression -->
  <arg name="colors"/>
  <arg value="sp"/>
</ruleset>
```

### Différences Standard Drupal vs PSR-12

| Critère | PSR-12 | Standard Drupal |
|---------|--------|-----------------|
| Indentation | 4 espaces | **2 espaces** |
| Accolade ouvrante de classe | Même ligne `{` | Même ligne `{` |
| Longueur de ligne | 120 chars | 80 chars recommandé |
| Commentaires | PHPDoc optionnel | PHPDoc **obligatoire** sur méthodes publiques |
| Type hints | Recommandé | Obligatoire |
| Blank lines après namespace | 1 | 1 |
| `use` statements | Groupés | Un par ligne |
| Espacement operateurs | ✅ | ✅ |

### Commandes

```bash
# Vérifier le code
docker compose exec php vendor/bin/phpcs \
  --standard=Drupal \
  web/modules/custom/mon_module/src

# Avec le phpcs.xml configuré (analyser tout le projet)
docker compose exec php vendor/bin/phpcs

# Voir uniquement les erreurs (pas les warnings)
docker compose exec php vendor/bin/phpcs \
  --standard=Drupal \
  --warning-severity=0 \
  web/modules/custom/mon_module/src

# Rapport détaillé avec suggestions de correction
docker compose exec php vendor/bin/phpcs \
  --standard=Drupal \
  --report=diff \
  web/modules/custom/mon_module/src/Service/ArticleService.php
```

---

## PHP Code Beautifier — Correction Automatique

`phpcbf` (PHP Code Beautifier and Fixer) corrige automatiquement les violations PHPCS corrigeables.

```bash
# Corriger tous les fichiers du module
docker compose exec php vendor/bin/phpcbf \
  --standard=Drupal \
  web/modules/custom/mon_module/src

# Corriger avec le phpcs.xml configuré (tout le projet)
docker compose exec php vendor/bin/phpcbf

# Corriger un fichier spécifique
docker compose exec php vendor/bin/phpcbf \
  --standard=Drupal \
  web/modules/custom/mon_module/src/Service/ArticleService.php
```

**Ce que phpcbf corrige automatiquement :**
- Indentation (4 espaces → 2 espaces)
- Espaces en fin de ligne
- Lignes vides superflues
- Espacement autour des opérateurs
- Ordre des modificateurs (`public`, `protected`, etc.)

**Ce que phpcbf NE peut PAS corriger :**
- Commentaires PHPDoc manquants
- Noms de variables/fonctions non conformes
- Logique de code incorrecte

---

## Drupal Check — Scanner les Dépréciations

`mglaman/drupal-check` scanne les modules pour détecter les APIs dépréciées **avant un upgrade**.

### Installation

```bash
composer require --dev mglaman/drupal-check
```

### Utilisation

```bash
# Scanner un module pour les dépréciations Drupal 10 → 11
docker compose exec php vendor/bin/drupal-check \
  web/modules/custom/mon_module

# Scanner avec le niveau de dépréciation (D9, D10, D11)
docker compose exec php vendor/bin/drupal-check \
  --drupal-root=web \
  web/modules/custom/mon_module

# Scanner tous les modules custom
docker compose exec php vendor/bin/drupal-check \
  web/modules/custom

# Format JSON (pour CI)
docker compose exec php vendor/bin/drupal-check \
  --format=json \
  web/modules/custom > deprecations.json
```

### Exemple de sortie

```
 ------ -----------------------------------------------------------------------
  Line   web/modules/custom/mon_module/src/Service/ArticleService.php
 ------ -----------------------------------------------------------------------
  42     \Drupal\Core\Entity\EntityManager is deprecated in drupal:9.0.0 and
         is removed from drupal:10.0.0. Use
         \Drupal\Core\Entity\EntityTypeManager instead.
         See https://www.drupal.org/node/2549139
 ------ -----------------------------------------------------------------------
```

---

## Rector pour Drupal — Refactoring Automatique

Rector applique automatiquement les migrations de code : annotations → attributs, APIs dépréciées → nouvelles APIs, PHP 7.x → PHP 8.x.

### Installation

```bash
composer require --dev palantirnet/drupal-rector
```

### `rector.php` — configuration complète

```php
<?php
// rector.php (à la racine du projet)

declare(strict_types=1);

use DrupalRector\Set\Drupal10SetList;
use DrupalRector\Set\Drupal11SetList;
use Rector\Config\RectorConfig;
use Rector\Set\ValueObject\LevelSetList;

return RectorConfig::configure()
  ->withPaths([
    __DIR__ . '/web/modules/custom',
    __DIR__ . '/web/themes/custom',
  ])
  ->withSets([
    // Migration Drupal 9 → 10
    Drupal10SetList::DRUPAL_10,
    // Migration Drupal 10 → 11
    Drupal11SetList::DRUPAL_11,
    // PHP 8.0 : match, named arguments, etc.
    LevelSetList::UP_TO_PHP_80,
    // PHP 8.1 : enums, readonly properties, etc.
    LevelSetList::UP_TO_PHP_81,
  ])
  ->withSkip([
    // Exclure les dossiers de tests si souhaité
    __DIR__ . '/web/modules/custom/*/tests',
  ]);
```

### Commandes

```bash
# Dry-run — voir les changements SANS les appliquer (obligatoire en CI)
docker compose exec php vendor/bin/rector process --dry-run

# Dry-run sur un module spécifique
docker compose exec php vendor/bin/rector process \
  web/modules/custom/mon_module \
  --dry-run

# Appliquer les corrections
docker compose exec php vendor/bin/rector process \
  web/modules/custom/mon_module

# Appliquer avec diff lisible
docker compose exec php vendor/bin/rector process \
  web/modules/custom/mon_module \
  --output-format=checkstyle

# Lister tous les sets Drupal disponibles
docker compose exec php vendor/bin/rector list-rules \
  --set=vendor/palantirnet/drupal-rector/config/drupal-10
```

### Exemple de transformation Rector (D10 → D11)

```php
// ❌ Avant Rector (D10, PHPUnit 10 annotations)
/**
 * @group mon_module
 * @covers \Drupal\mon_module\Service\MonService
 */
class MonTest extends UnitTestCase {}

// ✅ Après Rector (D11, PHPUnit 11 attributs PHP)
#[\PHPUnit\Framework\Attributes\Group('mon_module')]
#[\PHPUnit\Framework\Attributes\CoversClass(MonService::class)]
class MonTest extends UnitTestCase {}
```

---

## Pipeline GitLab CI Complet — Validate + Test + Coverage

```yaml
# .gitlab-ci.yml — pipeline complet Drupal avec analyse statique

stages:
  - validate
  - test
  - coverage

variables:
  COMPOSER_ALLOW_SUPERUSER: "1"
  COMPOSER_NO_INTERACTION: "1"
  # Base image PHP avec Drupal
  PHP_IMAGE: "drupal:11-php8.3-apache"
  # Base de données pour les tests fonctionnels
  MYSQL_ROOT_PASSWORD: "root"
  MYSQL_DATABASE: "drupal_test"
  MYSQL_USER: "drupal"
  MYSQL_PASSWORD: "drupal"

# ─────────────────────────────────────────────────────────
# STAGE 1 : VALIDATE — secondes, pas besoin de DB
# ─────────────────────────────────────────────────────────

validate:phpcs:
  stage: validate
  image: "${PHP_IMAGE}"
  before_script:
    - composer install --no-progress --prefer-dist --optimize-autoloader
  script:
    - docker compose exec php vendor/bin/phpcs
      # ou directement dans le container CI :
    - vendor/bin/phpcs --standard=Drupal web/modules/custom
  allow_failure: false
  artifacts:
    when: on_failure
    reports:
      codequality: phpcs-report.json
    paths:
      - phpcs-report.json

validate:phpstan:
  stage: validate
  image: "${PHP_IMAGE}"
  before_script:
    - composer install --no-progress --prefer-dist --optimize-autoloader
  script:
    - vendor/bin/phpstan analyse \
        --configuration=phpstan.neon \
        --no-progress \
        --error-format=gitlab \
        > phpstan-report.json || true
    - vendor/bin/phpstan analyse \
        --configuration=phpstan.neon \
        --no-progress
  artifacts:
    when: always
    reports:
      codequality: phpstan-report.json
  allow_failure: false

validate:rector:
  stage: validate
  image: "${PHP_IMAGE}"
  before_script:
    - composer install --no-progress --prefer-dist --optimize-autoloader
  script:
    # Dry-run uniquement — échoue si des migrations sont détectées
    - vendor/bin/rector process --dry-run --no-progress-bar
  allow_failure: false

# ─────────────────────────────────────────────────────────
# STAGE 2 : TEST — Unit + Kernel (SQLite) + Functional (MariaDB)
# ─────────────────────────────────────────────────────────

test:unit:
  stage: test
  image: "${PHP_IMAGE}"
  needs: ["validate:phpcs", "validate:phpstan", "validate:rector"]
  before_script:
    - composer install --no-progress --prefer-dist --optimize-autoloader
    - cp phpunit.xml.dist phpunit.xml
  script:
    # Unit tests : pas de DB nécessaire
    - vendor/bin/phpunit \
        --testsuite unit \
        --log-junit phpunit-unit.xml \
        --colors=never
  artifacts:
    reports:
      junit: phpunit-unit.xml
    when: always

test:kernel:
  stage: test
  image: "${PHP_IMAGE}"
  needs: ["validate:phpcs", "validate:phpstan", "validate:rector"]
  variables:
    # SQLite pour les Kernel tests — pas besoin de service MariaDB
    SIMPLETEST_DB: "sqlite://localhost/:memory:"
  before_script:
    - composer install --no-progress --prefer-dist --optimize-autoloader
    - cp phpunit.xml.dist phpunit.xml
  script:
    - vendor/bin/phpunit \
        --testsuite kernel \
        --log-junit phpunit-kernel.xml \
        --colors=never
  artifacts:
    reports:
      junit: phpunit-kernel.xml
    when: always

test:functional:
  stage: test
  image: "${PHP_IMAGE}"
  needs: ["test:unit", "test:kernel"]
  services:
    # MariaDB requis pour les Functional tests (installation Drupal complète)
    - name: mariadb:10.11
      alias: mariadb
  variables:
    SIMPLETEST_DB: "mysql://drupal:drupal@mariadb/drupal_test"
    SIMPLETEST_BASE_URL: "http://localhost"
    BROWSERTEST_OUTPUT_DIRECTORY: "/tmp/browser-output"
  before_script:
    - composer install --no-progress --prefer-dist --optimize-autoloader
    - cp phpunit.xml.dist phpunit.xml
    # Configurer Apache pour Drupal
    - a2enmod rewrite headers
    - service apache2 restart
    # Attendre que MariaDB soit prête
    - until mysqladmin ping -h mariadb --silent; do sleep 2; done
    # Créer la base de données de test
    - mysql -h mariadb -u root -proot -e "CREATE DATABASE IF NOT EXISTS drupal_test;"
    - mysql -h mariadb -u root -proot -e "GRANT ALL ON drupal_test.* TO 'drupal'@'%' IDENTIFIED BY 'drupal';"
  script:
    - vendor/bin/phpunit \
        --testsuite functional \
        --log-junit phpunit-functional.xml \
        --colors=never
  artifacts:
    reports:
      junit: phpunit-functional.xml
    paths:
      - /tmp/browser-output
    when: always
  when: on_success

# ─────────────────────────────────────────────────────────
# STAGE 3 : COVERAGE — PCOV + seuil minimum
# ─────────────────────────────────────────────────────────

coverage:
  stage: coverage
  image: "${PHP_IMAGE}"
  needs: ["test:unit", "test:kernel"]
  variables:
    SIMPLETEST_DB: "sqlite://localhost/:memory:"
  before_script:
    - composer install --no-progress --prefer-dist --optimize-autoloader
    - cp phpunit.xml.dist phpunit.xml
    # Activer PCOV (plus rapide que Xdebug pour la couverture)
    - docker-php-ext-enable pcov || true
  script:
    # Générer la couverture avec PCOV
    - php -d pcov.enabled=1 vendor/bin/phpunit \
        --testsuite unit \
        --coverage-text \
        --coverage-clover coverage.xml \
        --coverage-html coverage-html/ \
        --colors=never \
        --min-coverage=70
  coverage: '/^\s*Lines:\s*(\d+.\d+)\%/'
  artifacts:
    paths:
      - coverage.xml
      - coverage-html/
    when: always
    expire_in: 1 week
  allow_failure: false
```

### Utilisation locale (équivalent du pipeline CI)

```bash
# Étape 1 : validate
docker compose exec php vendor/bin/phpcs --standard=Drupal web/modules/custom
docker compose exec php vendor/bin/phpstan analyse --configuration=phpstan.neon --no-progress
docker compose exec php vendor/bin/rector process --dry-run --no-progress-bar

# Étape 2 : tests
docker compose exec php vendor/bin/phpunit --testsuite unit
docker compose exec php vendor/bin/phpunit --testsuite kernel
docker compose exec php vendor/bin/phpunit --testsuite functional

# Étape 3 : coverage
docker compose exec php php -d pcov.enabled=1 vendor/bin/phpunit \
  --testsuite unit \
  --coverage-html coverage-html/ \
  --min-coverage=70
```

---

## Mutation Testing avec Infection

Le mutation testing va plus loin que la couverture de code : il **modifie le code source** (mutations) et vérifie que les tests détectent ces modifications. Un test qui ne détecte pas la mutation est inutile, même s'il couvre la ligne.

**Exemples de mutations appliquées automatiquement :**
- `>` → `>=` ou `<`
- `true` → `false`
- `+` → `-`
- `return $value` → `return null`
- Supprimer un `if` entier

Un **MSI (Mutation Score Indicator) > 70%** signifie que les tests sont réellement efficaces.

### Installation

```bash
composer require --dev infection/infection
```

### Configuration `infection.json5`

```json5
// infection.json5 (à la racine du projet)
{
  "source": {
    "directories": ["web/modules/custom"],
    "excludes": [
      "web/modules/custom/*/tests",
      "web/modules/custom/*/node_modules"
    ]
  },
  "testFramework": "phpunit",
  "phpUnit": {
    "configDir": "."
  },
  "minMsi": 70,
  "minCoveredMsi": 80,
  "threads": 4,
  "logs": {
    "text": "infection.log",
    "html": "infection.html",
    "json": "infection-report.json"
  }
}
```

### Lancer Infection

```bash
# Analyse complète — avec seuils MSI
docker compose exec php vendor/bin/infection --min-msi=70 --min-covered-msi=80

# Verbose — voir chaque mutation appliquée
docker compose exec php vendor/bin/infection --min-msi=70 --show-mutations

# Sur un seul module
docker compose exec php vendor/bin/infection \
  --min-msi=70 \
  --filter=web/modules/custom/mon_module/src

# Avec 4 threads en parallèle (plus rapide)
docker compose exec php vendor/bin/infection --min-msi=70 --threads=4
```

### Résultats typiques

```
Infection — PHP Mutation Testing Framework
==========================================
Running initial test suite...

PHPUnit version: 10.5.x
Tests: 42, Assertions: 187, Time: 1.23s

Killed mutants:    156 / 220  (70.9%)
Escaped mutants:    48 / 220  (21.8%)
Timed out:           8 / 220   (3.6%)
Not covered:         8 / 220   (3.6%)

MSI:              70.9%   ← Mutation Score Indicator (seuil: 70%)
Covered MSI:      85.2%   ← MSI sur les lignes couvertes (seuil: 80%)

✅ The min MSI of 70% was reached.
✅ The min covered MSI of 80% was reached.
```

### Interpréter les résultats

| Terme | Signification | Action |
|-------|--------------|--------|
| **Killed** | La mutation a fait échouer un test | Bon signe — test efficace |
| **Escaped** | La mutation n'a pas été détectée | Écrire un test plus précis |
| **Timed out** | La mutation a causé une boucle infinie | Normal, compté comme tué |
| **Not covered** | Ligne non couverte par les tests | Écrire des tests pour cette ligne |

### Exemple — Mutation échappée et correction

```php
// Code original
public function isAdult(int $age): bool {
  return $age >= 18;
}

// Mutation appliquée par Infection : >= devient >
// public function isAdult(int $age): bool {
//   return $age > 18;  ← mutation
// }

// ❌ Test insuffisant — ne détecte pas la mutation
public function testIsAdult(): void {
  $this->assertTrue($this->service->isAdult(25));
  $this->assertFalse($this->service->isAdult(10));
}

// ✅ Test correct — teste exactement la limite
public function testIsAdult(): void {
  $this->assertTrue($this->service->isAdult(25));
  $this->assertTrue($this->service->isAdult(18));   // Exactement 18 = adulte
  $this->assertFalse($this->service->isAdult(17));  // 17 = mineur
  $this->assertFalse($this->service->isAdult(10));
}
```

### Pipeline GitLab CI — Mutation Testing

```yaml
# .gitlab-ci.yml — mutation testing (stage séparé, optionnel)
test:mutation:
  stage: test
  image: "${PHP_IMAGE}"
  needs: ["test:unit"]
  variables:
    SIMPLETEST_DB: "sqlite://localhost/:memory:"
  before_script:
    - composer install --no-progress --prefer-dist --optimize-autoloader
    - cp phpunit.xml.dist phpunit.xml
  script:
    - docker compose exec php vendor/bin/infection
        --min-msi=60
        --threads=4
        --no-progress
  artifacts:
    paths:
      - infection.log
      - infection.html
    when: always
    expire_in: 1 week
  allow_failure: true  # Ne bloque pas le pipeline — indicateur de qualité
  when: manual         # Déclenché manuellement (coûteux en temps)
  only:
    - main
    - merge_requests
```

> **Note :** Infection nécessite que PHPUnit ait généré une couverture de code (Xdebug ou PCOV activé). S'assurer que `php -d pcov.enabled=1` est utilisé pour la couverture, et qu'`infection.json5` pointe vers le bon `phpunit.xml`.
