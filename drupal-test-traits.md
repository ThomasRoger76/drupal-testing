# Drupal Test Traits (DTT) — Tester un Site Existant

## Quand Utiliser DTT

**Problème avec `BrowserTestBase` / `KernelTestBase` :** chaque test réinstalle Drupal from scratch → lent, et ne teste pas la vraie configuration de production.

**DTT résout ça :** tester contre une installation Drupal **existante** (ta vraie DB de dev/staging), sans réinstaller.

| | `KernelTestBase` | `BrowserTestBase` | **DTT `ExistingSiteBase`** |
|--|-----------------|------------------|--------------------------|
| Vitesse | Quelques secondes | 10-60s | **Millisecondes** |
| DB | Nouvelle DB de test | Nouvelle DB de test | **Vraie DB existante** |
| Config | Minimale | Installation complète | **Config réelle** |
| Données | Créées dans setUp() | Créées dans setUp() | **Données existantes** |
| Idéal pour | Services, hooks, Entity API | Formulaires, accès HTTP | **Smoke tests, contenu réel** |

**Quand utiliser DTT :**
- Vérifier qu'un type de contenu existe et a les bons champs
- Tester qu'une route répond correctement sur le vrai site
- Valider des permissions sur du contenu réel
- Smoke tests post-déploiement

**Quand NE PAS utiliser DTT :**
- Tester une logique métier isolée → Unit
- Tester des hooks Drupal → Kernel
- Tester des formulaires avec soumission → Functional

---

## Installation

```bash
composer require --dev weitzman/drupal-test-traits
```

### Structure du package

```
weitzman/drupal-test-traits
├── src/
│   ├── ExistingSiteBase.php         ← Classe de base principale
│   ├── ExistingSiteSelenium2DriverTestBase.php  ← Avec JS
│   ├── Entity/
│   │   └── NodeCreationTrait.php    ← Créer des nodes de test
│   └── GovernmentTestTrait.php      ← Helpers supplémentaires
```

---

## Configuration `phpunit.xml` pour DTT

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit
  bootstrap="vendor/autoload.php"
  colors="true"
  verbose="true"
  failOnWarning="true">

  <testsuites>
    <!-- Tests PHPUnit standard Drupal -->
    <testsuite name="unit">
      <directory>web/modules/custom/*/tests/src/Unit</directory>
    </testsuite>
    <testsuite name="kernel">
      <directory>web/modules/custom/*/tests/src/Kernel</directory>
    </testsuite>
    <testsuite name="functional">
      <directory>web/modules/custom/*/tests/src/Functional</directory>
    </testsuite>

    <!-- Tests DTT — site existant -->
    <testsuite name="existing-site">
      <directory>web/modules/custom/*/tests/src/ExistingSite</directory>
    </testsuite>
  </testsuites>

  <php>
    <!-- DB du VRAI site (pas une DB de test) -->
    <env name="DTT_BASE_URL" value="http://localhost"/>
    <!-- Pour les tests avec browser (ExistingSiteSelenium2DriverTestBase) -->
    <env name="DTT_MINK_DRIVER_ARGS" value='["chrome", {"browserName":"chrome","goog:chromeOptions":{"args":["--disable-gpu","--headless","--no-sandbox"]}}, "http://selenium:4444/wd/hub"]'/>

    <!-- DB Drupal existante (même que settings.php) -->
    <env name="SIMPLETEST_DB" value="mysql://drupal:drupal@localhost/drupal_dev"/>
    <env name="SIMPLETEST_BASE_URL" value="http://localhost"/>
  </php>

</phpunit>
```

---

## Structure de Base — `ExistingSiteBase`

```php
<?php
// web/modules/custom/mon_module/tests/src/ExistingSite/MonSmokeTest.php
namespace Drupal\Tests\mon_module\ExistingSite;

use weitzman\DrupalTestTraits\ExistingSiteBase;

#[\PHPUnit\Framework\Attributes\Group('existing-site')]
#[\PHPUnit\Framework\Attributes\Group('mon_module')]
final class MonSmokeTest extends ExistingSiteBase {
  // Pas de $modules, pas de setUp() compliqué
  // Le site existant est utilisé tel quel
}
```

---

## Exemple 1 — Vérifier qu'un Node Existe

```php
<?php
namespace Drupal\Tests\mon_module\ExistingSite;

use Drupal\node\Entity\Node;
use weitzman\DrupalTestTraits\ExistingSiteBase;

#[\PHPUnit\Framework\Attributes\Group('existing-site')]
final class ContentSmokeTest extends ExistingSiteBase {

  /**
   * Vérifie que la page d'accueil existe et est publiée.
   */
  public function testPageAccueilExiste(): void {
    $nodes = \Drupal::entityTypeManager()
      ->getStorage('node')
      ->loadByProperties([
        'type'   => 'page',
        'title'  => 'Accueil',
        'status' => 1,
      ]);

    $this->assertNotEmpty($nodes, "La page d'accueil doit exister et être publiée.");
  }

  /**
   * Vérifie qu'au moins 10 articles publiés existent.
   */
  public function testArticlesPubliesExistent(): void {
    $nids = \Drupal::entityTypeManager()
      ->getStorage('node')
      ->getQuery()
      ->condition('type', 'article')
      ->condition('status', 1)
      ->accessCheck(FALSE)
      ->count()
      ->execute();

    $this->assertGreaterThanOrEqual(10, $nids,
      "Le site doit avoir au moins 10 articles publiés."
    );
  }

  /**
   * Vérifie que le type de contenu 'article' a bien le champ image.
   */
  public function testTypeContenuArticleAChampImage(): void {
    $field_definitions = \Drupal::service('entity_field.manager')
      ->getFieldDefinitions('node', 'article');

    $this->assertArrayHasKey('field_image', $field_definitions,
      "Le type de contenu 'article' doit avoir le champ field_image."
    );
    $this->assertEquals(
      'image',
      $field_definitions['field_image']->getType(),
      "field_image doit être de type 'image'."
    );
  }
}
```

---

## Exemple 2 — Tester une Route

```php
<?php
namespace Drupal\Tests\mon_module\ExistingSite;

use weitzman\DrupalTestTraits\ExistingSiteBase;

#[\PHPUnit\Framework\Attributes\Group('existing-site')]
final class RoutesSmokeTest extends ExistingSiteBase {

  /**
   * Vérifie que les pages principales répondent HTTP 200.
   */
  #[\PHPUnit\Framework\Attributes\DataProvider('routesPubliquesProvider')]
  public function testRoutesPubliquesAccessibles(string $path): void {
    $this->drupalGet($path);
    $this->assertSession()->statusCodeEquals(200);
  }

  public static function routesPubliquesProvider(): array {
    return [
      'page d\'accueil'  => ['/'],
      'page recherche'   => ['/search'],
      'plan du site'     => ['/sitemap'],
    ];
  }

  /**
   * Vérifie que les routes admin sont protégées.
   */
  #[\PHPUnit\Framework\Attributes\DataProvider('routesAdminProvider')]
  public function testRoutesAdminProtegees(string $path): void {
    // On est anonyme (pas de drupalLogin)
    $this->drupalGet($path);
    // Doit retourner 403 ou rediriger vers /user/login
    $code = $this->getSession()->getStatusCode();
    $this->assertContains($code, [403, 302],
      "La route $path doit être protégée pour les anonymes."
    );
  }

  public static function routesAdminProvider(): array {
    return [
      'admin'          => ['/admin'],
      'admin/config'   => ['/admin/config'],
      'admin/content'  => ['/admin/content'],
    ];
  }

  /**
   * Vérifie l'accès admin avec les droits.
   */
  public function testRouteAdminAvecDroits(): void {
    // Créer un utilisateur admin temporaire pour ce test
    $admin = $this->createUser(['access administration pages', 'administer nodes']);
    $this->drupalLogin($admin);

    $this->drupalGet('/admin/content');
    $this->assertSession()->statusCodeEquals(200);
    $this->assertSession()->pageTextContains('Content');
  }
}
```

---

## Exemple 3 — Tester des Permissions sur Contenu Réel

```php
<?php
namespace Drupal\Tests\mon_module\ExistingSite;

use Drupal\node\Entity\Node;
use weitzman\DrupalTestTraits\ExistingSiteBase;

#[\PHPUnit\Framework\Attributes\Group('existing-site')]
final class PermissionsSmokeTest extends ExistingSiteBase {

  /**
   * Vérifie qu'un utilisateur anonyme ne peut pas accéder au contenu premium.
   */
  public function testContenuPremiumBloqueAnonyme(): void {
    // Trouver un node premium existant sur le vrai site
    $nodes = \Drupal::entityTypeManager()
      ->getStorage('node')
      ->loadByProperties([
        'type'            => 'article',
        'field_premium'   => 1,
        'status'          => 1,
      ]);

    if (empty($nodes)) {
      $this->markTestSkipped("Pas de contenu premium sur ce site — test ignoré.");
    }

    $node = reset($nodes);
    $this->drupalGet('/node/' . $node->id());

    // Le contenu premium doit bloquer les anonymes
    $this->assertSession()->statusCodeEquals(403);
  }

  /**
   * Vérifie qu'un éditeur peut modifier son propre contenu.
   */
  public function testEditeurModifiesonContenu(): void {
    // Créer un utilisateur éditeur TEMPORAIRE pour ce test
    $editeur = $this->createUser([
      'create article content',
      'edit own article content',
    ]);
    $this->drupalLogin($editeur);

    // Créer un node de test (sera nettoyé après le test)
    $node = $this->createNode([
      'type'   => 'article',
      'title'  => 'Test DTT — à supprimer',
      'status' => 1,
      'uid'    => $editeur->id(),
    ]);

    $this->drupalGet('/node/' . $node->id() . '/edit');
    $this->assertSession()->statusCodeEquals(200);
    $this->assertSession()->fieldExists('title[0][value]');
  }

  /**
   * Vérifie qu'un visiteur connecté ne peut pas modifier le contenu des autres.
   */
  public function testVisiteurNePeutPasModifierAutresContenu(): void {
    // Trouver un node existant sur le vrai site
    $nids = \Drupal::entityTypeManager()
      ->getStorage('node')
      ->getQuery()
      ->condition('type', 'article')
      ->condition('status', 1)
      ->range(0, 1)
      ->accessCheck(FALSE)
      ->execute();

    if (empty($nids)) {
      $this->markTestSkipped("Pas d'article sur ce site.");
    }

    $nid = reset($nids);

    // Créer un utilisateur sans permission d'édition
    $visiteur = $this->createUser(['access content']);
    $this->drupalLogin($visiteur);

    $this->drupalGet('/node/' . $nid . '/edit');
    $this->assertSession()->statusCodeEquals(403);
  }
}
```

---

## Nettoyage Automatique

DTT nettoie automatiquement les entités créées via `createNode()`, `createUser()`, etc. — les données du vrai site ne sont **pas touchées**.

```php
// Ces entités sont créées pour le test ET supprimées automatiquement après
$user = $this->createUser(['access content']);
$node = $this->createNode(['type' => 'article', 'title' => 'Test temporaire']);

// Les données existantes du site ne sont JAMAIS modifiées par DTT
// (sauf si tu appelles directement ->save() ou ->delete() sur des entités existantes)
```

---

## Lancer les Tests DTT

```bash
# Lancer uniquement les tests DTT (site existant)
docker compose exec php vendor/bin/phpunit --testsuite existing-site

# Lancer un groupe spécifique
docker compose exec php vendor/bin/phpunit --group existing-site

# Lancer un fichier DTT spécifique
docker compose exec php vendor/bin/phpunit \
  web/modules/custom/mon_module/tests/src/ExistingSite/RoutesSmokeTest.php

# Lancer tous les tests (DTT + PHPUnit standard)
docker compose exec php vendor/bin/phpunit
```

---

## Avantages DTT vs BrowserTestBase

| Critère | BrowserTestBase | DTT ExistingSiteBase |
|---------|-----------------|----------------------|
| Vitesse | 10-60s par test | **< 1s par test (10×+)** |
| DB utilisée | Nouvelle DB de test | **Vraie DB du site** |
| Config testée | Minimale | **Config réelle de prod** |
| Données | Créées synthétiquement | **Données réelles** |
| Détecte les problèmes de config | ❌ | ✅ |
| Détecte les bugs de données | ❌ | ✅ |
| Idéal pour post-déploiement | ❌ | **✅ Smoke tests** |
| Isolation totale | ✅ | ❌ (partage la DB) |
| Reproductible 100% | ✅ | Dépend des données |

**Stratégie recommandée :** combiner les deux
- PHPUnit standard (Unit/Kernel/Functional) → CI sur chaque commit
- DTT → smoke tests post-déploiement en staging/prod
