# Tester le Layer Thème (CSS / Templates / JS)

Une mise à jour de `hook_preprocess_node`, une classe CSS retirée, ou une library qui ne se charge plus peut passer complètement inaperçue sans tests. Ce fichier couvre les patterns pour détecter ces régressions.

---

## Tester les Classes CSS dans le HTML Rendu (Functional)

```php
use Drupal\Tests\BrowserTestBase;

final class ThemeRenderTest extends BrowserTestBase {
  protected static $modules = ['mon_module', 'node'];
  protected $defaultTheme = 'stark';

  public function testClassesCSSNodeArticle(): void {
    $node = $this->drupalCreateNode(['type' => 'article', 'status' => 1]);
    $this->drupalGet('/node/' . $node->id());

    // Vérifier la classe de type sur l'article
    $this->assertSession()->elementExists('css', 'article.node--type-article');
    $this->assertSession()->elementExists('css', 'article.node--view-mode-full');

    // Vérifier une classe dynamique ajoutée par preprocess
    $this->assertSession()->elementExists('css', 'article.node--has-image');

    // Vérifier une classe BEM sur un champ
    $this->assertSession()->elementExists('css', '.field--name-field-image');
    $this->assertSession()->elementExists('css', '.field--type-image');
  }

  public function testClassesCSSNoeudNonPublie(): void {
    $node = $this->drupalCreateNode(['type' => 'article', 'status' => 0]);
    $admin = $this->drupalCreateUser(['bypass node access']);
    $this->drupalLogin($admin);

    $this->drupalGet('/node/' . $node->id());

    // La classe d'état "non publié" doit être présente
    $this->assertSession()->elementExists('css', 'article.node--unpublished');
    $this->assertSession()->elementNotExists('css', 'article.node--promoted');
  }

  public function testAttributesDynamiques(): void {
    $node = $this->drupalCreateNode(['type' => 'article', 'status' => 1]);
    $this->drupalGet('/node/' . $node->id());

    // Vérifier un attribut data-* ou aria-*
    $this->assertSession()->elementAttributeContains('css', '.node', 'data-node-id', (string) $node->id());

    // Vérifier loading="lazy" sur les images
    $this->assertSession()->elementAttributeContains('css', '.field--name-field-image img', 'loading', 'lazy');
  }
}
```

---

## Tester les Preprocess Variables (Kernel)

```php
use Drupal\KernelTests\KernelTestBase;
use Drupal\node\Entity\Node;
use Drupal\node\Entity\NodeType;

final class PreprocessTest extends KernelTestBase {
  protected static $modules = ['mon_module', 'node', 'user', 'field', 'text', 'filter', 'system'];

  protected function setUp(): void {
    parent::setUp();
    $this->installEntitySchema('node');
    $this->installEntitySchema('user');
    $this->installConfig(['filter', 'node']);
    $this->installSchema('system', ['sequences']);
    NodeType::create(['type' => 'article', 'name' => 'Article'])->save();
  }

  public function testPreprocessNodeAjouteVariable(): void {
    $node = Node::create([
      'type'   => 'article',
      'title'  => 'Mon article',
      'status' => 1,
      'uid'    => 0,
    ]);
    $node->save();

    // Simuler les variables Twig en appelant le preprocess
    $variables = [
      'node'       => $node,
      'view_mode'  => 'full',
      'elements'   => ['#node' => $node, '#view_mode' => 'full'],
      'attributes' => new \Drupal\Core\Template\Attribute(),
    ];

    // Appeler la fonction preprocess du module
    mon_module_preprocess_node($variables);

    // Vérifier les variables injectées
    $this->assertArrayHasKey('date_formatted', $variables);
    $this->assertNotEmpty($variables['date_formatted']);
    $this->assertArrayHasKey('is_featured', $variables);
  }

  public function testPreprocessInjecteClasses(): void {
    $node = Node::create(['type' => 'article', 'title' => 'Test', 'status' => 1, 'uid' => 0]);
    $node->save();

    $attributes = new \Drupal\Core\Template\Attribute();
    $variables = [
      'node'       => $node,
      'view_mode'  => 'teaser',
      'attributes' => $attributes,
      'elements'   => ['#node' => $node, '#view_mode' => 'teaser'],
    ];

    mon_module_preprocess_node($variables);

    // Vérifier qu'une classe custom a été ajoutée
    $this->assertTrue($variables['attributes']->hasClass('node--teaser-custom'));
  }
}
```

---

## Tester qu'une Library CSS/JS est Chargée (Functional)

```php
public function testLibraryGlobaleChargee(): void {
  $this->drupalGet('<front>');

  // Vérifier qu'un fichier CSS de la library est dans la source de la page
  $html = $this->getSession()->getPage()->getContent();
  $this->assertStringContainsString('mon_theme/css/components/button.css', $html);
}

public function testLibraryConditionnelleChargee(): void {
  $node = $this->drupalCreateNode(['type' => 'article', 'status' => 1]);
  $this->drupalGet('/node/' . $node->id());

  $html = $this->getSession()->getPage()->getContent();
  // La library "article-styles" ne doit charger QUE sur les pages article
  $this->assertStringContainsString('mon_theme/css/article.css', $html);
}

public function testLibraryAbsenteSurAutresPages(): void {
  // Sur une page "page" (non article)
  $page = $this->drupalCreateNode(['type' => 'page', 'status' => 1]);
  $this->drupalGet('/node/' . $page->id());

  $html = $this->getSession()->getPage()->getContent();
  $this->assertStringNotContainsString('css/article.css', $html);
}
```

---

## Tester les Render Arrays (Kernel)

```php
public function testRenderArrayContientClasses(): void {
  $node = Node::create(['type' => 'article', 'title' => 'Test', 'status' => 1, 'uid' => 0]);
  $node->save();

  $view_builder = $this->container->get('entity_type.manager')->getViewBuilder('node');
  $build = $view_builder->view($node, 'teaser');

  // Rendre le tableau en HTML
  $rendered = (string) $this->container->get('renderer')->renderRoot($build);

  $this->assertStringContainsString('node--type-article', $rendered);
  $this->assertStringContainsString('node--view-mode-teaser', $rendered);
}

public function testRenderArrayFieldFormatter(): void {
  $node = Node::create([
    'type'         => 'article',
    'title'        => 'Test formatter',
    'status'       => 1,
    'uid'          => 0,
    'field_image'  => NULL,
  ]);
  $node->save();

  $display = \Drupal::entityTypeManager()
    ->getStorage('entity_view_display')
    ->load('node.article.default');

  $build = $display->buildMultiple([$node]);
  $rendered = (string) $this->container->get('renderer')->renderRoot($build[0]);

  // Vérifier le markup produit par le formatter
  $this->assertStringContainsString('field--name-title', $rendered);
}
```

---

## Tester les Suggestions de Template (Functional avec Twig Debug)

```php
public function testSuggestionTemplateUtilisee(): void {
  // Activer Twig debug uniquement pour ce test
  \Drupal::configFactory()
    ->getEditable('system.performance')
    ->set('twig.cache', FALSE)
    ->save();

  $node = $this->drupalCreateNode(['type' => 'article', 'status' => 1]);
  $this->drupalGet('/node/' . $node->id());

  // Vérifier le template utilisé via la présence de son markup spécifique
  // (plus fiable que parser les commentaires HTML qui dépendent du debug mode)
  // Astuce : ajouter un attribut data-template dans node--article.html.twig
  $this->assertSession()->elementAttributeContains(
    'css', 'article.node',
    'data-template',
    'node--article'
  );
}
```

---

## Tester l'Output HTML de Blocs (Functional)

```php
public function testBlocRenduAvecClasses(): void {
  $this->drupalGet('<front>');

  // Vérifier la structure d'un bloc
  $this->assertSession()->elementExists('css', '.block--system-branding-block');
  $this->assertSession()->elementExists('css', '.site-name');
  $this->assertSession()->elementAttributeContains('css', '.site-logo img', 'alt', '');
}

public function testBlocAbsentSiConditionNonRemplie(): void {
  // Tester qu'un bloc avec condition d'affichage ne s'affiche pas
  $this->drupalGet('/node/add/article');
  $this->assertSession()->elementNotExists('css', '.block--conditions-bloc');
}
```

---

## Tester les Twig Extensions Custom (Unit)

```php
use Drupal\Tests\UnitTestCase;

final class MonTwigExtensionTest extends UnitTestCase {

  private \Drupal\mon_module\TwigExtension\MonExtension $extension;

  protected function setUp(): void {
    parent::setUp();
    $this->extension = new \Drupal\mon_module\TwigExtension\MonExtension();
  }

  public function testFiltreFormatDate(): void {
    $timestamp = mktime(0, 0, 0, 1, 15, 2026);
    $result = $this->extension->formatDate($timestamp, 'd/m/Y');
    $this->assertEquals('15/01/2026', $result);
  }

  public function testFonctionTruncate(): void {
    $result = $this->extension->truncate('Lorem ipsum dolor sit amet', 10);
    $this->assertEquals('Lorem ipsu…', $result);
  }
}
```
