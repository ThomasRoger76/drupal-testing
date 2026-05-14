# Scénarios Avancés — Email, Queue, Cron, Views, HTTP, Migrations...

## Traits de Création — Raccourcis Indispensables

Drupal fournit des Traits qui simplifient massivement la création de données de test.

```php
use Drupal\Tests\node\Traits\ContentTypeCreationTrait;
use Drupal\Tests\node\Traits\NodeCreationTrait;
use Drupal\Tests\user\Traits\UserCreationTrait;
use Drupal\Tests\taxonomy\Traits\TaxonomyTestTrait;

final class MonTest extends KernelTestBase {
  use ContentTypeCreationTrait;
  use NodeCreationTrait;
  use UserCreationTrait;

  protected function setUp(): void {
    parent::setUp();
    $this->installEntitySchema('node');
    $this->installEntitySchema('user');
    $this->installSchema('system', ['sequences']);

    // Créer un Content Type en une ligne
    $this->createContentType(['type' => 'article', 'name' => 'Article']);

    // Créer un nœud en une ligne (sans gérer le storage manuellement)
    $node = $this->createNode([
      'type'   => 'article',
      'title'  => 'Mon article',
      'status' => 1,
    ]);

    // Créer un utilisateur avec permissions
    $user = $this->createUser(['create article content', 'edit own article content']);
  }
}
```

---

## Tester les Emails

```php
use Drupal\KernelTests\KernelTestBase;

final class EmailTest extends KernelTestBase {
  protected static $modules = ['mon_module', 'system', 'user', 'node'];

  protected function setUp(): void {
    parent::setUp();
    $this->installEntitySchema('user');
    $this->installSchema('system', ['sequences']);

    // Activer le collecteur de mails pour les tests
    \Drupal::configFactory()
      ->getEditable('system.mail')
      ->set('interface.default', 'test_mail_collector')
      ->save();
  }

  public function testEmailEnvoye(): void {
    // Déclencher l'action qui envoie l'email
    $service = $this->container->get('mon_module.notification_service');
    $service->envoyerNotification('user@example.com', 'Sujet test', 'Corps du message');

    // Récupérer les emails capturés
    $emails = \Drupal::state()->get('system.test_mail_collector') ?? [];

    $this->assertCount(1, $emails, 'Un seul email doit être envoyé');
    $this->assertEquals('user@example.com', $emails[0]['to']);
    $this->assertStringContainsString('Sujet test', $emails[0]['subject']);
    $this->assertStringContainsString('Corps du message', $emails[0]['body']);
  }

  public function testEmailNonEnvoyeSiConditionNonRemplie(): void {
    $service = $this->container->get('mon_module.notification_service');
    $service->envoyerNotification('', 'Sujet', 'Corps');  // email vide

    $emails = \Drupal::state()->get('system.test_mail_collector') ?? [];
    $this->assertEmpty($emails, 'Aucun email ne doit être envoyé sans destinataire valide');
  }

  protected function tearDown(): void {
    // Nettoyer les emails capturés entre les tests
    \Drupal::state()->delete('system.test_mail_collector');
    parent::tearDown();
  }
}
```

---

## Tester les QueueWorkers

```php
use Drupal\KernelTests\KernelTestBase;

final class QueueWorkerTest extends KernelTestBase {
  protected static $modules = ['mon_module', 'system', 'node', 'user', 'field', 'text'];

  protected function setUp(): void {
    parent::setUp();
    $this->installSchema('system', ['sequences', 'queue']);
    $this->installEntitySchema('node');
    $this->installEntitySchema('user');
  }

  public function testQueueWorkerProcesseItem(): void {
    // Ajouter un item dans la queue
    $queue = \Drupal::queue('mon_module_traitement');
    $queue->createItem(['node_id' => 42, 'action' => 'process']);
    $this->assertEquals(1, $queue->numberOfItems());

    // Instancier et exécuter le worker
    /** @var \Drupal\mon_module\Plugin\QueueWorker\MonTraitementWorker $worker */
    $worker = $this->container->get('plugin.manager.queue_worker')
      ->createInstance('mon_module_traitement');

    $item = $queue->claimItem();
    $worker->processItem($item->data);
    $queue->deleteItem($item);

    $this->assertEquals(0, $queue->numberOfItems());

    // Vérifier l'effet du traitement
    $state = \Drupal::state()->get('mon_module.processed_items', []);
    $this->assertContains(42, $state);
  }

  public function testQueueWorkerGereException(): void {
    $worker = $this->container->get('plugin.manager.queue_worker')
      ->createInstance('mon_module_traitement');

    // Les items invalides doivent lever SuspendQueueException ou être ignorés
    $this->expectException(\Drupal\Core\Queue\SuspendQueueException::class);
    $worker->processItem(['node_id' => null]);
  }
}
```

---

## Tester le Cron (`hook_cron`)

```php
use Drupal\KernelTests\KernelTestBase;

final class CronTest extends KernelTestBase {
  protected static $modules = ['mon_module', 'system', 'user'];

  protected function setUp(): void {
    parent::setUp();
    $this->installSchema('system', ['sequences']);
  }

  public function testCronMiseAJourState(): void {
    // État initial
    $this->assertNull(\Drupal::state()->get('mon_module.last_cron'));

    // Exécuter le cron
    \Drupal::service('cron')->run();

    // Vérifier que hook_cron() a été exécuté
    $last_cron = \Drupal::state()->get('mon_module.last_cron');
    $this->assertNotNull($last_cron);
    $this->assertGreaterThan(0, $last_cron);
  }

  public function testCronNettoyeAnciennesEntrees(): void {
    // Créer des données périmées
    \Drupal::state()->set('mon_module.entries', [
      ['date' => time() - 86400 * 31, 'data' => 'vieux'],
      ['date' => time(),               'data' => 'récent'],
    ]);

    \Drupal::service('cron')->run();

    $entries = \Drupal::state()->get('mon_module.entries', []);
    $this->assertCount(1, $entries, 'Les entrées vieilles de 31+ jours doivent être nettoyées');
    $this->assertEquals('récent', $entries[0]['data']);
  }
}
```

---

## Tester les EventSubscribers (Unit + Kernel)

### Unit — Logique Pure

```php
use Drupal\Tests\UnitTestCase;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\HttpKernelInterface;

final class MonSubscriberTest extends UnitTestCase {

  public function testOnRequestRedirectionAnonymePourAdmin(): void {
    $current_user = $this->createMock(\Drupal\Core\Session\AccountProxyInterface::class);
    $current_user->method('isAnonymous')->willReturn(TRUE);

    $subscriber = new \Drupal\mon_module\EventSubscriber\MonSubscriber($current_user);

    $kernel  = $this->createMock(HttpKernelInterface::class);
    $request = Request::create('/admin/structure');
    $event   = new RequestEvent($kernel, $request, HttpKernelInterface::MAIN_REQUEST);

    $subscriber->onRequest($event);

    // L'utilisateur anonyme sur /admin doit être redirigé
    $this->assertTrue($event->hasResponse());
    $response = $event->getResponse();
    $this->assertInstanceOf(\Symfony\Component\HttpFoundation\RedirectResponse::class, $response);
  }

  public function testOnRequestPasDeRedirectionUtilisateurConnecte(): void {
    $current_user = $this->createMock(\Drupal\Core\Session\AccountProxyInterface::class);
    $current_user->method('isAnonymous')->willReturn(FALSE);

    $subscriber = new \Drupal\mon_module\EventSubscriber\MonSubscriber($current_user);

    $kernel  = $this->createMock(HttpKernelInterface::class);
    $request = Request::create('/admin/structure');
    $event   = new RequestEvent($kernel, $request, HttpKernelInterface::MAIN_REQUEST);

    $subscriber->onRequest($event);

    $this->assertFalse($event->hasResponse());
  }
}
```

---

## Tester les Field Formatters et Widgets (Kernel)

```php
use Drupal\KernelTests\KernelTestBase;
use Drupal\node\Entity\Node;
use Drupal\Tests\node\Traits\ContentTypeCreationTrait;

final class FieldFormatterTest extends KernelTestBase {
  use ContentTypeCreationTrait;

  protected static $modules = ['mon_module', 'node', 'user', 'field', 'field_ui', 'text', 'filter', 'system'];

  protected function setUp(): void {
    parent::setUp();
    $this->installEntitySchema('node');
    $this->installEntitySchema('user');
    $this->installConfig(['filter', 'field', 'node']);
    $this->installSchema('system', ['sequences']);
    $this->createContentType(['type' => 'article', 'name' => 'Article']);
  }

  public function testFormatterRenduHTML(): void {
    $node = Node::create([
      'type'        => 'article',
      'title'       => 'Test',
      'status'      => 1,
      'uid'         => 0,
      'field_texte' => 'Valeur du champ',
    ]);
    $node->save();

    // Configurer le formatter dans le display
    $display = \Drupal::entityTypeManager()
      ->getStorage('entity_view_display')
      ->load('node.article.default');

    $display->setComponent('field_texte', [
      'type'     => 'mon_module_custom_formatter',
      'settings' => ['prefix' => '>>'],
    ])->save();

    // Rendre le nœud
    $view_builder = \Drupal::entityTypeManager()->getViewBuilder('node');
    $build        = $view_builder->view($node, 'default');
    $rendered     = (string) \Drupal::service('renderer')->renderRoot($build);

    $this->assertStringContainsString('>>Valeur du champ', $rendered);
    $this->assertStringContainsString('field--type-mon-module', $rendered);
  }
}
```

---

## Tester les Views (Functional + Kernel)

### Functional — Tester l'affichage d'une View

```php
use Drupal\Tests\BrowserTestBase;
use Drupal\Tests\node\Traits\NodeCreationTrait;

final class ViewsTest extends BrowserTestBase {
  use NodeCreationTrait;

  protected static $modules = ['mon_module', 'node', 'views', 'views_ui'];
  protected $defaultTheme = 'stark';

  protected function setUp(): void {
    parent::setUp();
    // Créer des données de test
    $this->createNode(['type' => 'article', 'title' => 'Article 1', 'status' => 1]);
    $this->createNode(['type' => 'article', 'title' => 'Article 2', 'status' => 1]);
    $this->createNode(['type' => 'article', 'title' => 'Brouillon',  'status' => 0]);
  }

  public function testViewAfficheArticlesPublies(): void {
    $this->drupalGet('/mes-articles');
    $this->assertSession()->statusCodeEquals(200);
    $this->assertSession()->elementsCount('css', '.views-row', 2);
    $this->assertSession()->pageTextContains('Article 1');
    $this->assertSession()->pageTextContains('Article 2');
    $this->assertSession()->pageTextNotContains('Brouillon');
  }

  public function testViewFiltreExposeFonctionne(): void {
    $this->drupalGet('/mes-articles');
    $this->submitForm(['title' => 'Article 1'], 'Appliquer');
    $this->assertSession()->elementsCount('css', '.views-row', 1);
    $this->assertSession()->pageTextContains('Article 1');
    $this->assertSession()->pageTextNotContains('Article 2');
  }
}
```

### Kernel — Tester une View programmatiquement

```php
use Drupal\KernelTests\KernelTestBase;
use Drupal\views\Tests\ViewResultAssertionTrait;

final class ViewsKernelTest extends KernelTestBase {
  use ViewResultAssertionTrait;

  protected static $modules = ['mon_module', 'node', 'user', 'views', 'field', 'text', 'filter', 'system'];

  protected function setUp(): void {
    parent::setUp();
    $this->installEntitySchema('node');
    $this->installEntitySchema('user');
    $this->installConfig(['views', 'node', 'filter']);
    $this->installSchema('system', ['sequences']);
  }

  public function testViewRetourneResultats(): void {
    // Créer des nœuds
    \Drupal\node\Entity\Node::create(['type' => 'article', 'title' => 'A', 'status' => 1, 'uid' => 0])->save();
    \Drupal\node\Entity\Node::create(['type' => 'article', 'title' => 'B', 'status' => 1, 'uid' => 0])->save();

    $view = \Drupal\views\Views::getView('mon_module_articles');
    $this->assertNotNull($view, 'La View mon_module_articles doit exister');

    $view->execute('page_1');
    $this->assertCount(2, $view->result);
  }
}
```

---

## Tester les Appels HTTP Externes (Mocking GuzzleHttp)

```php
use Drupal\KernelTests\KernelTestBase;
use GuzzleHttp\Client;
use GuzzleHttp\Handler\MockHandler;
use GuzzleHttp\HandlerStack;
use GuzzleHttp\Psr7\Response;

final class ExternalApiTest extends KernelTestBase {
  protected static $modules = ['mon_module', 'system'];

  public function testAppelApiExterneSucces(): void {
    // Créer un client HTTP mocké
    $mock = new MockHandler([
      new Response(200, ['Content-Type' => 'application/json'], json_encode([
        'status' => 'ok',
        'data'   => ['id' => 42, 'name' => 'Item A'],
      ])),
    ]);
    $client = new Client(['handler' => HandlerStack::create($mock)]);

    // Injecter le client mocké dans le service
    $service = new \Drupal\mon_module\Service\ExternalApiService($client);
    $result  = $service->fetchItem(42);

    $this->assertEquals('Item A', $result['name']);
  }

  public function testAppelApiExtternEchec404(): void {
    $mock   = new MockHandler([new Response(404, [], 'Not Found')]);
    $client = new Client(['handler' => HandlerStack::create($mock)]);

    $service = new \Drupal\mon_module\Service\ExternalApiService($client);

    $this->expectException(\Drupal\mon_module\Exception\ApiException::class);
    $service->fetchItem(999);
  }
}
```

---

## Tester l'Upload de Fichiers (Functional)

```php
use Drupal\Tests\BrowserTestBase;

final class FileUploadTest extends BrowserTestBase {
  protected static $modules = ['mon_module', 'node', 'file', 'image'];
  protected $defaultTheme = 'stark';

  public function testUploadImage(): void {
    $user = $this->drupalCreateUser(['create article content']);
    $this->drupalLogin($user);

    $this->drupalGet('/node/add/article');

    // Créer un fichier image de test en mémoire
    $image_path = \Drupal::service('file_system')->getTempDirectory() . '/test-image-' . uniqid() . '.png';
    // Créer une vraie image PNG minimale
    $image = imagecreatetruecolor(10, 10);
    imagepng($image, $image_path);
    imagedestroy($image);

    // Attacher le fichier au champ
    $this->getSession()->getPage()->attachFileToField('files[field_image_0]', $image_path);
    $this->assertSession()->assertWaitOnAjaxRequest();

    // Vérifier l'aperçu
    $this->assertSession()->elementExists('css', '.image-preview img');

    // Soumettre le formulaire
    $this->submitForm(['title[0][value]' => 'Article avec image'], 'Enregistrer');
    $this->assertSession()->pageTextContains('Article avec image');
    $this->assertSession()->elementExists('css', '.field--name-field-image img');

    // Nettoyer
    unlink($image_path);
  }
}
```

---

## Tester le Multilingue (Kernel)

```php
use Drupal\KernelTests\KernelTestBase;
use Drupal\language\Entity\ConfigurableLanguage;
use Drupal\node\Entity\Node;
use Drupal\Tests\node\Traits\ContentTypeCreationTrait;

final class MultilingualTest extends KernelTestBase {
  use ContentTypeCreationTrait;

  protected static $modules = [
    'mon_module', 'node', 'user', 'field', 'text', 'filter', 'system',
    'language', 'content_translation',
  ];

  protected function setUp(): void {
    parent::setUp();
    $this->installEntitySchema('node');
    $this->installEntitySchema('user');
    $this->installConfig(['filter', 'language', 'content_translation']);
    $this->installSchema('system', ['sequences']);
    $this->createContentType(['type' => 'article', 'name' => 'Article']);

    // Ajouter la langue anglaise
    ConfigurableLanguage::createFromLangcode('en')->save();
  }

  public function testNoeudTraduit(): void {
    $node = Node::create([
      'type'     => 'article',
      'title'    => 'Titre en français',
      'status'   => 1,
      'uid'      => 0,
      'langcode' => 'fr',
    ]);
    $node->save();

    // Ajouter une traduction anglaise
    $node->addTranslation('en', ['title' => 'English title'])->save();

    // Vérifier les traductions
    $this->assertTrue($node->hasTranslation('en'));
    $this->assertEquals('English title', $node->getTranslation('en')->label());
    $this->assertEquals('Titre en français', $node->getTranslation('fr')->label());
  }
}
```

---

## Tester les Migrations (Kernel)

```php
use Drupal\Tests\migrate\Kernel\MigrateTestBase;

final class MonMigrationTest extends MigrateTestBase {
  protected static $modules = ['mon_module', 'migrate', 'node', 'user', 'field', 'text', 'filter', 'system'];

  protected function setUp(): void {
    parent::setUp();
    $this->installEntitySchema('node');
    $this->installEntitySchema('user');
    $this->installConfig(['filter', 'node']);
    $this->installSchema('system', ['sequences']);

    // Configurer les données source pour la migration
    $this->database->schema()->createTable('source_articles', [
      'fields' => [
        'id'    => ['type' => 'int', 'unsigned' => TRUE, 'not null' => TRUE],
        'title' => ['type' => 'varchar', 'length' => 255],
        'body'  => ['type' => 'text'],
      ],
      'primary key' => ['id'],
    ]);

    $this->database->insert('source_articles')->fields([
      ['id' => 1, 'title' => 'Article migré 1', 'body' => 'Corps 1'],
      ['id' => 2, 'title' => 'Article migré 2', 'body' => 'Corps 2'],
    ])->execute();
  }

  public function testMigrationImporteArticles(): void {
    $this->executeMigration('mon_module_articles');

    $nodes = \Drupal::entityTypeManager()->getStorage('node')
      ->loadByProperties(['type' => 'article']);

    $this->assertCount(2, $nodes);

    $titles = array_map(fn($n) => $n->label(), $nodes);
    $this->assertContains('Article migré 1', $titles);
    $this->assertContains('Article migré 2', $titles);
  }
}
```

---

## Tester les Endpoints REST / JSON API (Functional)

```php
use Drupal\Tests\BrowserTestBase;

final class RestApiTest extends BrowserTestBase {
  protected static $modules = ['mon_module', 'node', 'serialization', 'rest', 'basic_auth'];
  protected $defaultTheme = 'stark';

  public function testGetEndpointRequiertAuthentification(): void {
    $this->drupalGet('/api/articles', ['query' => ['_format' => 'json']]);
    $this->assertSession()->statusCodeEquals(401);
  }

  public function testGetEndpointAvecAuth(): void {
    $user = $this->drupalCreateUser(['restful get entity:node']);
    $this->drupalLogin($user);

    $this->createNode(['type' => 'article', 'title' => 'Via REST', 'status' => 1]);

    $this->drupalGet('/api/articles', ['query' => ['_format' => 'json']]);
    $this->assertSession()->statusCodeEquals(200);

    $data = json_decode($this->getSession()->getPage()->getContent(), TRUE);
    $this->assertIsArray($data);
    $this->assertNotEmpty($data);
  }

  public function testPostCreeeNoeud(): void {
    $user = $this->drupalCreateUser(['restful post entity:node', 'create article content']);

    // Utiliser Basic Auth
    $credentials = base64_encode($user->getAccountName() . ':' . $user->passRaw);
    $this->drupalGet('/entity/node', [
      'query'   => ['_format' => 'json'],
    ]);
    $this->assertSession()->statusCodeEquals(200);  // Adapter selon l'endpoint
  }

  public function testPatchModifieNoeud(): void {
    $user = $this->drupalCreateUser(['edit own article content', 'restful patch entity:node']);
    $node = $this->createNode([
      'type'   => 'article',
      'title'  => 'Original',
      'uid'    => $user->id(),
      'status' => 1,
    ]);

    $this->drupalLogin($user);
    // Test PATCH selon la configuration REST du module
    $this->drupalGet('/node/' . $node->id() . '?_format=json');
    $this->assertSession()->statusCodeEquals(200);
    $data = json_decode($this->getSession()->getPage()->getContent(), TRUE);
    $this->assertEquals('Original', $data['title'][0]['value']);
  }
}
```

---

## Nightwatch.js — Tests JS Style Drupal Core

Drupal core utilise Nightwatch.js pour ses propres tests JavaScript.

```bash
# Prérequis : ChromeDriver + Node.js
cd web
npm install

# Lancer les tests Nightwatch d'un module
yarn nightwatch --tag mon_module
# OU
node_modules/.bin/nightwatch --config core/tests/Nightwatch/nightwatch.conf.js \
  modules/custom/mon_module/tests/src/Nightwatch/
```

```javascript
// web/modules/custom/mon_module/tests/src/Nightwatch/MonBehaviorTest.js
module.exports = {
  '@tags': ['mon_module'],

  'Mon comportement AJAX fonctionne': (browser) => {
    browser
      .drupalInstall({ installProfile: 'standard' })
      .drupalInstallModule(['mon_module'])
      .drupalLogin({ name: 'admin', password: 'admin' })
      .drupalRelativeURL('/node/add/article')
      .waitForElementPresent('.node-article-form', 5000)
      .click('#mon-bouton-ajax')
      .waitForElementPresent('.ajax-result', 5000)
      .assert.containsText('.ajax-result', 'Succès')
      .drupalUninstallModule('mon_module')
      .drupalUninstall()
      .end();
  },
};
```

---

## Jest — Tester `Drupal.behaviors` en Isolation

```bash
# package.json à la racine
npm install --save-dev jest @testing-library/dom jest-environment-jsdom
```

```javascript
// jest.config.js
module.exports = {
  testEnvironment: 'jsdom',
  testMatch: ['**/tests/src/Jest/**/*.test.js'],
};
```

```javascript
// web/modules/custom/mon_module/tests/src/Jest/mon-behavior.test.js
const { readFileSync } = require('fs');

// Charger le comportement à tester
const behaviorCode = readFileSync(
  './web/modules/custom/mon_module/js/mon-comportement.js', 'utf8'
);

describe('Drupal.behaviors.monComportement', () => {
  let container;

  beforeEach(() => {
    // Setup de l'environnement DOM
    document.body.innerHTML = '';
    container = document.createElement('div');
    document.body.appendChild(container);

    // Mock de l'objet Drupal global
    global.Drupal = { behaviors: {}, t: (str) => str };
    global.once = jest.fn((name, selector, ctx) =>
      Array.from((ctx || document).querySelectorAll(selector))
    );

    // Évaluer le code du behavior
    eval(behaviorCode);
  });

  afterEach(() => jest.clearAllMocks());

  test('attach lie un event listener sur .mon-selecteur', () => {
    container.innerHTML = '<button class="mon-bouton">Click</button>';
    const mockHandler = jest.fn();

    // Surcharger la fonction interne pour le test
    jest.spyOn(container.querySelector('.mon-bouton'), 'addEventListener');

    Drupal.behaviors.monComportement.attach(container, {});

    expect(once).toHaveBeenCalledWith('mon-comportement', '.mon-bouton', container);
  });

  test('une fois attaché, le comportement ne se re-lie pas (once)', () => {
    container.innerHTML = '<button class="mon-bouton">Click</button>';
    Drupal.behaviors.monComportement.attach(container, {});
    Drupal.behaviors.monComportement.attach(container, {}); // 2ème appel

    // once() garantit l'unicité
    expect(once).toHaveBeenCalledTimes(2); // appelé 2 fois mais ne re-bind pas
  });
});
```

```bash
# Lancer les tests Jest
npx jest --testPathPattern="mon_module"
```
