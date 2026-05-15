# Kernel Tests — `KernelTestBase`

## Quand Utiliser

- Interaction avec l'Entity API (créer, charger, modifier des entités)
- Interaction avec la Config API
- Tester des hooks Drupal (`hook_entity_presave`, etc.)
- Tester des services qui nécessitent un vrai container Drupal
- Tester des EventSubscribers ou des plugins
- Durée : **quelques secondes** — plus lent que Unit, beaucoup plus rapide que Functional

**Concept :** Drupal est partiellement instancié (container + DB minimale) mais pas "installé" comme un vrai site.

---

## Structure de Base

```php
<?php
// web/modules/custom/mon_module/tests/src/Kernel/ArticleServiceKernelTest.php
namespace Drupal\Tests\mon_module\Kernel;

use Drupal\KernelTests\KernelTestBase;
use Drupal\node\Entity\Node;
use Drupal\node\Entity\NodeType;

// ❌ D8/D9/D10 (annotations docblock — supprimées dans PHPUnit 11)
// /** @group mon_module */

// ✅ D11 / PHPUnit 11 (attributs PHP — standard)
#[\PHPUnit\Framework\Attributes\Group('mon_module')]
final class ArticleServiceKernelTest extends KernelTestBase {

  /**
   * Modules à activer pour ce test — UNIQUEMENT ceux nécessaires.
   */
  protected static $modules = [
    'mon_module',
    'node',
    'user',
    'field',
    'text',
    'filter',
    'system',
  ];

  protected function setUp(): void {
    parent::setUp();

    // Monter le schéma de base
    $this->installSchema('system', ['sequences']);
    $this->installSchema('node', ['node_access']);

    // Créer les tables des entités nécessaires
    $this->installEntitySchema('node');
    $this->installEntitySchema('user');

    // Importer la config par défaut des modules
    $this->installConfig(['node', 'field', 'mon_module']);

    // Créer un Content Type pour les tests
    NodeType::create(['type' => 'article', 'name' => 'Article'])->save();
  }
}
```

---

## Méthodes `install*` — Référence Complète

```php
// Créer les tables d'une entité
$this->installEntitySchema('node');
$this->installEntitySchema('user');
$this->installEntitySchema('taxonomy_term');
$this->installEntitySchema('media');

// Créer les tables d'un schéma DB d'un module
$this->installSchema('system', ['sequences']);
$this->installSchema('node', ['node_access']);
$this->installSchema('mon_module', ['mon_module_items']);

// Importer la config par défaut d'un module (config/install/*.yml)
$this->installConfig(['node']);
$this->installConfig(['mon_module']);
$this->installConfig(['field', 'text', 'filter']);  // pour les champs texte

// Attention : l'ordre compte — les dépendances d'abord
```

---

## Tester l'Entity API

```php
public function testCreerEtChargerNode(): void {
  // Créer un nœud
  $node = Node::create([
    'type'   => 'article',
    'title'  => 'Mon article de test',
    'status' => 1,
    'uid'    => 0,  // User 0 (anonymous) pour les tests simples
  ]);
  $node->save();

  $nid = $node->id();
  $this->assertGreaterThan(0, $nid);

  // Charger depuis la DB
  $loaded = Node::load($nid);
  $this->assertNotNull($loaded);
  $this->assertEquals('Mon article de test', $loaded->label());
  $this->assertTrue($loaded->isPublished());
}

public function testModifierNode(): void {
  $node = Node::create(['type' => 'article', 'title' => 'Titre original', 'status' => 1]);
  $node->save();

  $node->set('title', 'Titre modifié');
  $node->save();

  $reloaded = Node::load($node->id());
  $this->assertEquals('Titre modifié', $reloaded->label());
}

public function testSupprimerNode(): void {
  $node = Node::create(['type' => 'article', 'title' => 'À supprimer', 'status' => 1]);
  $node->save();
  $nid = $node->id();

  $node->delete();

  $this->assertNull(Node::load($nid));
}
```

---

## Tester la Config API

```php
public function testConfigDefaults(): void {
  // $this->config() est disponible dans KernelTestBase
  $config = $this->config('mon_module.settings');

  // Vérifie les valeurs par défaut (depuis config/install/mon_module.settings.yml)
  $this->assertEquals(10, $config->get('max_items'));
  $this->assertEquals('basic', $config->get('mode'));
}

public function testModifierConfig(): void {
  // $this->config() retourne ImmutableConfig — utiliser getEditable() pour écrire
  \Drupal::configFactory()
    ->getEditable('mon_module.settings')
    ->set('max_items', 25)
    ->save();

  // Relire — $this->config() pour la lecture
  $this->assertEquals(25, $this->config('mon_module.settings')->get('max_items'));
}
```

---

## Tester des Services

```php
public function testMonServiceDepuisContainer(): void {
  // Accéder au service via le container
  $service = $this->container->get('mon_module.article_service');
  $this->assertInstanceOf(ArticleServiceInterface::class, $service);

  // Créer les données nécessaires
  $node = Node::create(['type' => 'article', 'title' => 'Test', 'status' => 1]);
  $node->save();

  // Appeler le service
  $result = $service->getRecentArticles(5);
  $this->assertCount(1, $result);
  $this->assertEquals('Test', reset($result)->label());
}
```

---

## Tester des Hooks Drupal

```php
/**
 * Test que hook_entity_presave() du module ajoute un préfixe au titre.
 */
public function testHookEntityPresaveAjoutePrefixe(): void {
  // Le hook est déclenché automatiquement lors du save
  $node = Node::create([
    'type'   => 'article',
    'title'  => 'Mon titre',
    'status' => 1,
  ]);
  $node->save();

  // Le hook doit avoir modifié le titre
  $this->assertStringStartsWith('[Article] ', $node->label());
}

/**
 * Test que hook_node_access() bloque l'accès selon les conditions.
 */
public function testHookNodeAccessBlocageSiNonPublie(): void {
  $node = Node::create(['type' => 'article', 'title' => 'Brouillon', 'status' => 0]);
  $node->save();

  $account = $this->container->get('current_user');
  $access = \Drupal::entityTypeManager()
    ->getAccessControlHandler('node')
    ->access($node, 'view', $account);

  $this->assertFalse($access);
}
```

---

## Tester une EntityQuery

```php
public function testEntityQueryAvecConditions(): void {
  // Créer plusieurs nœuds
  Node::create(['type' => 'article', 'title' => 'A', 'status' => 1])->save();
  Node::create(['type' => 'article', 'title' => 'B', 'status' => 0])->save();
  Node::create(['type' => 'page',    'title' => 'C', 'status' => 1])->save();

  $nids = \Drupal::entityTypeManager()
    ->getStorage('node')
    ->getQuery()
    ->condition('type', 'article')
    ->condition('status', 1)
    ->accessCheck(FALSE)
    ->execute();

  $this->assertCount(1, $nids);
}
```

---

## Tester des Formulaires (Kernel level)

```php
use Drupal\Core\Form\FormState;

public function testBuildForm(): void {
  $form_object = $this->container->get('form_builder')
    ->getForm('Drupal\mon_module\Form\MonForm');

  $this->assertArrayHasKey('max_items', $form_object);
  $this->assertEquals('number', $form_object['max_items']['#type']);
}

public function testSubmitForm(): void {
  $form_state = new FormState();
  $form_state->setValue('max_items', 25);

  $form_builder = $this->container->get('form_builder');
  $form_builder->submitForm('Drupal\mon_module\Form\SettingsForm', $form_state);

  // Vérifier que la config a été mise à jour
  $this->assertEquals(25, $this->config('mon_module.settings')->get('max_items'));
}
```

---

## Modules Courants à Inclure dans `$modules`

```php
protected static $modules = [
  // Base systèmes — presque toujours nécessaires
  'system',
  'user',

  // Pour les nœuds et champs
  'node',
  'field',
  'field_ui',
  'text',
  'filter',

  // Pour les fichiers et médias
  'file',
  'image',
  'media',

  // Pour les taxonomies
  'taxonomy',

  // Pour les formulaires complexes
  'options',

  // Pour les Views en Kernel
  'views',

  // Token support
  'token',

  // Module à tester
  'mon_module',
];
```

**Règle :** commencer avec le minimum. Ajouter des modules seulement si une erreur `requires module X` apparaît.

---

## Créer des Utilisateurs dans les Kernel Tests

```php
use Drupal\user\Entity\User;
use Drupal\user\Entity\Role;

protected function createTestUser(array $permissions = []): User {
  // Créer un rôle avec les permissions nécessaires
  $role = Role::create(['id' => 'test_role_' . uniqid(), 'label' => 'Test Role']);
  foreach ($permissions as $permission) {
    $role->grantPermission($permission);
  }
  $role->save();

  $user = User::create([
    'name'   => 'test_user_' . uniqid(),
    'status' => 1,
    'roles'  => [$role->id()],
  ]);
  $user->save();

  return $user;
}
```
