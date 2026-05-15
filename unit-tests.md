# Unit Tests — `UnitTestCase`

## Quand Utiliser

- Logique PHP pure sans dépendances Drupal (calculs, validations, formatage)
- Services qui peuvent avoir leurs dépendances **mockées**
- Helpers, utilitaires, algorithmes
- Durée : quelques **millisecondes** — le type de test le plus rapide

**Ne pas utiliser pour :** accéder à la DB, charger des entités, appeler des services Drupal réels.

---

## Structure de Base

```php
<?php
// web/modules/custom/mon_module/tests/src/Unit/PrixCalculatorTest.php
namespace Drupal\Tests\mon_module\Unit;

use Drupal\mon_module\Service\PrixCalculator;
use Drupal\Tests\UnitTestCase;

// ❌ D8/D9/D10 (annotations docblock — supprimées dans PHPUnit 11)
// /**
//  * @group mon_module
//  * @coversDefaultClass \Drupal\mon_module\Service\PrixCalculator
//  */

// ✅ D11 / PHPUnit 11 (attributs PHP — standard)
#[\PHPUnit\Framework\Attributes\Group('mon_module')]
#[\PHPUnit\Framework\Attributes\CoversClass(PrixCalculator::class)]
final class PrixCalculatorTest extends UnitTestCase {

  private PrixCalculator $calculator;

  protected function setUp(): void {
    parent::setUp();
    $this->calculator = new PrixCalculator();
  }

  /**
   * @covers ::calculer
   */
  public function testCalculerPrixAvecTaxe(): void {
    $result = $this->calculator->calculer(100.0, 0.20);
    $this->assertEquals(120.0, $result);
  }

  public function testPrixZeroRetourneZero(): void {
    $this->assertEquals(0.0, $this->calculator->calculer(0.0, 0.20));
  }
}
```

---

## Mocking — Simuler des Services Drupal

Le mocking permet de tester un service qui dépend d'autres services, sans vraiment les instancier.

```php
<?php
namespace Drupal\Tests\mon_module\Unit;

use Drupal\Core\Entity\EntityStorageInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\mon_module\Service\ArticleService;
use Drupal\node\NodeInterface;
use Drupal\Tests\UnitTestCase;

// ❌ D8/D9/D10 (annotations docblock — supprimées dans PHPUnit 11)
// /**
//  * @group mon_module
//  */

// ✅ D11 / PHPUnit 11 (attributs PHP — standard)
#[\PHPUnit\Framework\Attributes\Group('mon_module')]
final class ArticleServiceTest extends UnitTestCase {

  private ArticleService $service;
  private EntityTypeManagerInterface $entityTypeManager;

  protected function setUp(): void {
    parent::setUp();

    // Créer un mock de NodeInterface
    $node = $this->createMock(NodeInterface::class);
    $node->method('label')->willReturn('Mon article de test');
    $node->method('bundle')->willReturn('article');
    $node->method('id')->willReturn(42);
    $node->method('isPublished')->willReturn(TRUE);

    // Créer un mock du storage
    $storage = $this->createMock(EntityStorageInterface::class);
    $storage->method('load')
      ->with(42)
      ->willReturn($node);
    $storage->method('loadMultiple')
      ->willReturn([$node]);

    // Créer un mock de EntityTypeManager
    $this->entityTypeManager = $this->createMock(EntityTypeManagerInterface::class);
    $this->entityTypeManager->method('getStorage')
      ->with('node')
      ->willReturn($storage);

    // Instancier le service avec les mocks
    $this->service = new ArticleService($this->entityTypeManager);
  }

  public function testGetArticleLabel(): void {
    $label = $this->service->getArticleLabel(42);
    $this->assertEquals('Mon article de test', $label);
  }

  public function testGetArticleInexistantRetourneNull(): void {
    // Le mock retourne null pour un ID inconnu
    $storage = $this->entityTypeManager->getStorage('node');
    // Reconfigurer le mock pour cet ID
    $this->entityTypeManager->method('getStorage')
      ->willReturn($this->createMock(EntityStorageInterface::class));

    $result = $this->service->getArticleLabel(999);
    $this->assertNull($result);
  }
}
```

### Méthodes de mock utiles

```php
// Retourner une valeur fixe
$mock->method('nomMethode')->willReturn($valeur);

// Retourner des valeurs différentes selon le paramètre
$mock->method('get')->willReturnMap([
  ['key_a', 'value_a'],
  ['key_b', 'value_b'],
]);

// Retourner une valeur selon le callback
$mock->method('nomMethode')->willReturnCallback(function($arg) {
  return $arg * 2;
});

// Lever une exception
$mock->method('nomMethode')->willThrowException(new \RuntimeException('Erreur'));

// Vérifier que la méthode est appelée exactement N fois
$mock->expects($this->exactly(2))
  ->method('save');

// Vérifier que la méthode est appelée avec des paramètres précis
$mock->expects($this->once())
  ->method('set')
  ->with($this->equalTo('status'), $this->equalTo(1));

// Vérifier que la méthode n'est JAMAIS appelée
$mock->expects($this->never())->method('delete');
```

---

## Data Providers — Tester Plusieurs Cas

```php
// ❌ D8/D9/D10 (annotations docblock — supprimées dans PHPUnit 11)
// /**
//  * @dataProvider prixProvider
//  * @covers ::calculer
//  */

// ✅ D11 / PHPUnit 11 (attributs PHP — standard)
#[\PHPUnit\Framework\Attributes\DataProvider('prixProvider')]
#[\PHPUnit\Framework\Attributes\CoversMethod('calculer')]
public function testCalculerPrix(float $base, float $taux, float $expected): void {
  $this->assertEquals($expected, $this->calculator->calculer($base, $taux));
}

/**
 * Fournit les cas de test pour testCalculerPrix.
 *
 * @return array<string, array{float, float, float}>
 */
public static function prixProvider(): array {
  return [
    'prix normal avec TVA 20%'  => [100.0, 0.20, 120.0],
    'prix zéro'                  => [0.0,   0.20, 0.0],
    'taux zéro'                  => [100.0, 0.0,  100.0],
    'taux élevé'                 => [50.0,  0.50, 75.0],
    'petits montants'            => [1.0,   0.05, 1.05],
  ];
}
```

---

## Tester les Exceptions

```php
public function testPrixNegatifLeveException(): void {
  $this->expectException(\InvalidArgumentException::class);
  $this->expectExceptionMessage('Le prix ne peut pas être négatif');

  $this->calculator->calculer(-100.0, 0.20);
}

public function testTauxSuperieurA1LeveException(): void {
  $this->expectException(\RangeException::class);
  $this->calculator->calculer(100.0, 1.5);
}
```

---

## Mocker `StringTranslationTrait` (`$this->t()`)

Beaucoup de services Drupal utilisent `$this->t()`. `UnitTestCase` fournit `getStringTranslationStub()` :

```php
protected function setUp(): void {
  parent::setUp();

  $service = new MonService();
  // Injecter le stub de traduction pour que $this->t() fonctionne
  $service->setStringTranslation($this->getStringTranslationStub());
  $this->service = $service;
}
```

---

## Assertions Utiles

```php
// Égalité stricte (type + valeur)
$this->assertSame('article', $type);

// Égalité non stricte
$this->assertEquals(42, $count);

// Null
$this->assertNull($result);
$this->assertNotNull($result);

// Booléens
$this->assertTrue($isPublished);
$this->assertFalse($isDeleted);

// Tableaux
$this->assertCount(3, $items);
$this->assertContains('valeur', $array);
$this->assertArrayHasKey('key', $array);
$this->assertEmpty($result);
$this->assertNotEmpty($result);

// Strings
$this->assertStringContainsString('needle', $haystack);
$this->assertStringStartsWith('prefix', $string);
$this->assertMatchesRegularExpression('/pattern/', $string);

// Types
$this->assertInstanceOf(NodeInterface::class, $node);
$this->assertIsArray($result);
$this->assertIsString($value);
```

---

## Tester des DTOs et des Value Objects

Les DTOs (Data Transfer Objects) et les objets de validation sont les candidats parfaits aux Unit Tests — logique pure PHP, aucune dépendance Drupal.

```php
<?php
// tests/src/Unit/DTO/ItemDTOTest.php
namespace Drupal\Tests\mon_module\Unit\DTO;

use Drupal\mon_module\DTO\ItemDTO;
use Drupal\Tests\UnitTestCase;

// ❌ D8/D9/D10 (annotations docblock — supprimées dans PHPUnit 11)
// /**
//  * @group mon_module
//  * @coversDefaultClass \Drupal\mon_module\DTO\ItemDTO
//  */

// ✅ D11 / PHPUnit 11 (attributs PHP — standard)
#[\PHPUnit\Framework\Attributes\Group('mon_module')]
#[\PHPUnit\Framework\Attributes\CoversClass(ItemDTO::class)]
final class ItemDTOTest extends UnitTestCase {

  // ❌ D10- : /** @covers ::fromArray @dataProvider itemProvider */
  // ✅ D11 / PHPUnit 11
  #[\PHPUnit\Framework\Attributes\DataProvider('itemProvider')]
  #[\PHPUnit\Framework\Attributes\CoversMethod('fromArray')]
  public function testFromArray(array $data, string $expectedId, bool $expectedValid): void {
    $dto = ItemDTO::fromArray($data);
    $this->assertEquals($expectedId, $dto->externalId);
    $this->assertEquals($expectedValid, $dto->isValid());
  }

  public static function itemProvider(): array {
    return [
      'valid item'         => [['id' => '123', 'name' => 'Test', 'status' => 'active'], '123', TRUE],
      'missing id'         => [['name' => 'Test'], '', FALSE],
      'empty name'         => [['id' => '123', 'name' => ''], '123', FALSE],
      'null status'        => [['id' => '456', 'name' => 'Other'], '456', TRUE],
    ];
  }
}
```

## Tester un Validator PHP Pur

```php
<?php
// tests/src/Unit/Validator/ItemValidatorTest.php
namespace Drupal\Tests\mon_module\Unit\Validator;

use Drupal\mon_module\DTO\ItemDTO;
use Drupal\mon_module\Validator\ItemValidator;
use Drupal\Tests\UnitTestCase;

// ❌ D8/D9/D10 (annotations docblock — supprimées dans PHPUnit 11)
// /** @group mon_module */

// ✅ D11 / PHPUnit 11 (attributs PHP — standard)
#[\PHPUnit\Framework\Attributes\Group('mon_module')]
final class ItemValidatorTest extends UnitTestCase {

  private ItemValidator $validator;

  protected function setUp(): void {
    parent::setUp();
    $this->validator = new ItemValidator();
  }

  public function testIsValidReturnsTrueForActiveItem(): void {
    $dto = ItemDTO::fromArray(['id' => '1', 'name' => 'Crèche Test', 'status' => 'validé']);
    $this->assertTrue($this->validator->isValid($dto));
  }

  public function testIsValidReturnsFalseForInvalidStatus(): void {
    $dto = ItemDTO::fromArray(['id' => '1', 'name' => 'Crèche Test', 'status' => 'fermé']);
    $this->assertFalse($this->validator->isValid($dto));
  }

  public function testIsIgnoredWithFutureStartDate(): void {
    $dto = ItemDTO::fromArray([
      'id'        => '1',
      'name'      => 'Test',
      'status'    => 'validé',
      'startDate' => (new \DateTime('+1 year'))->format('Y-m-d'),
    ]);
    $this->assertTrue($this->validator->isIgnored($dto));
  }

  public function testIsIgnoredReturnsFalseWhenNoStartDate(): void {
    $dto = ItemDTO::fromArray(['id' => '1', 'name' => 'Test', 'status' => 'validé']);
    $this->assertFalse($this->validator->isIgnored($dto));
  }
}
```

**Pourquoi tester en Unit et pas Kernel :** les DTOs et Validators sont de la **logique PHP pure** — pas de DB, pas de services Drupal. Quelques millisecondes par test, feedback instantané.

---

## Tester un Service avec `\Drupal::` — Pattern Partiel

Si le service utilise `\Drupal::` (procédural), utiliser `$this->createMock()` avec un container :

```php
protected function setUp(): void {
  parent::setUp();

  // Pour les services qui utilisent \Drupal::service() en interne
  // Préférer refactorer le service pour utiliser DI à la place
  // Si impossible, utiliser ProphecyTrait ou le container de test
  $container = $this->getMockBuilder(ContainerInterface::class)->getMock();
  $container->method('get')
    ->with('mon_service')
    ->willReturn($this->createMock(MonServiceInterface::class));

  \Drupal::setContainer($container);
}

protected function tearDown(): void {
  // parent::tearDown() gère le cleanup du container Drupal
  parent::tearDown();
  // Ne PAS faire \Drupal::setContainer(new ContainerBuilder()) — container vide invalide
}
```

**Règle :** si tu te retrouves à mocker `\Drupal::`, refactore le service pour utiliser l'injection de dépendances — le test te le dit.
