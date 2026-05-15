# Functional Tests — `BrowserTestBase`

## Quand Utiliser

- Tester l'accès HTTP (codes 200, 403, 404, 302)
- Tester la navigation utilisateur (admin vs anonyme vs rôle spécifique)
- Tester la soumission de formulaires avec validation
- Tester les endpoints REST/JSON API
- Tester des redirections
- Durée : **10-60 secondes par test** — Drupal complet installe en DB de test

---

## Structure de Base

```php
<?php
// web/modules/custom/mon_module/tests/src/Functional/MonPageTest.php
namespace Drupal\Tests\mon_module\Functional;

use Drupal\Tests\BrowserTestBase;
use Drupal\Tests\node\Traits\NodeCreationTrait;
use Drupal\Tests\node\Traits\ContentTypeCreationTrait;

// ❌ D8/D9/D10 (annotations docblock — supprimées dans PHPUnit 11)
// /** @group mon_module */

// ✅ D11 / PHPUnit 11 (attributs PHP — standard)
#[\PHPUnit\Framework\Attributes\Group('mon_module')]
final class MonPageTest extends BrowserTestBase {
  use NodeCreationTrait;
  use ContentTypeCreationTrait;

  protected static $modules = ['mon_module', 'node'];

  // OBLIGATOIRE — sans ça, le test échoue avec "theme not found"
  protected $defaultTheme = 'stark';

  protected function setUp(): void {
    parent::setUp();
    // Ici Drupal est complètement installé
    // Créer les Content Types, la config, les utilisateurs...
  }
}
```

---

## Navigation et Assertions HTTP

```php
public function testPageAccesibleAnonyme(): void {
  // Accéder à une URL (utilisateur anonyme par défaut)
  $this->drupalGet('/mon-module/page');
  $this->assertSession()->statusCodeEquals(200);
}

public function testPageInterditeAnonyme(): void {
  $this->drupalGet('/admin/config/mon-module');
  // Redirigé vers login → 403 ou 302
  $this->assertSession()->statusCodeEquals(403);
}

public function testPageInexistante(): void {
  $this->drupalGet('/url-qui-nexiste-pas');
  $this->assertSession()->statusCodeEquals(404);
}

public function testRedirection(): void {
  $this->drupalGet('/ancienne-url');
  // Vérifier qu'on a été redirigé
  $this->assertSession()->addressEquals('/nouvelle-url');
}
```

---

## Gestion des Utilisateurs

```php
public function testAccesAdminAvecPermission(): void {
  // Créer un utilisateur avec permissions spécifiques
  $user = $this->drupalCreateUser([
    'administer mon module',
    'access administration pages',
  ]);
  $this->drupalLogin($user);

  $this->drupalGet('/admin/config/mon-module/settings');
  $this->assertSession()->statusCodeEquals(200);
  $this->assertSession()->pageTextContains('Paramètres Mon Module');
}

public function testAccesRefuseSansPermission(): void {
  // Utilisateur sans la permission
  $user = $this->drupalCreateUser(['access content']);
  $this->drupalLogin($user);

  $this->drupalGet('/admin/config/mon-module/settings');
  $this->assertSession()->statusCodeEquals(403);
}

public function testSuperAdmin(): void {
  // Utilisateur #1 (bypass all permissions)
  $admin = $this->drupalCreateUser([], NULL, TRUE);
  $this->drupalLogin($admin);

  $this->drupalGet('/admin');
  $this->assertSession()->statusCodeEquals(200);
}

public function testDeconnexion(): void {
  $user = $this->drupalCreateUser(['access content']);
  $this->drupalLogin($user);
  $this->drupalLogout();

  // Tenter d'accéder à une page protégée après logout
  $this->drupalGet('/user/' . $user->id() . '/edit');
  $this->assertSession()->statusCodeEquals(403);
}
```

---

## Soumettre des Formulaires

```php
public function testSoumettreFormulaireSettings(): void {
  $admin = $this->drupalCreateUser(['administer mon module']);
  $this->drupalLogin($admin);

  $this->drupalGet('/admin/config/mon-module/settings');
  $this->assertSession()->statusCodeEquals(200);

  // Soumettre le formulaire
  $this->submitForm(
    [
      'max_items'    => 25,
      'mode'         => 'advanced',
      'enable_feature' => TRUE,
    ],
    'Enregistrer la configuration'  // Texte du bouton submit
  );

  // Vérifier le message de confirmation
  $this->assertSession()->pageTextContains('Configuration saved');

  // Vérifier que la config a été sauvegardée
  $config = $this->config('mon_module.settings');
  $this->assertEquals(25, $config->get('max_items'));
}

public function testValidationFormulaire(): void {
  $admin = $this->drupalCreateUser(['administer mon module']);
  $this->drupalLogin($admin);

  $this->drupalGet('/admin/config/mon-module/settings');
  $this->submitForm(
    ['max_items' => -5],   // Valeur invalide
    'Enregistrer la configuration'
  );

  // Le formulaire doit afficher une erreur de validation
  $this->assertSession()->pageTextContains('La valeur doit être supérieure à 0');
  // Et ne PAS avoir sauvegardé
  $this->assertSession()->pageTextNotContains('Configuration saved');
}

public function testCreerNoeud(): void {
  $user = $this->drupalCreateUser(['create article content']);
  $this->drupalLogin($user);

  $this->drupalGet('/node/add/article');
  $this->submitForm(
    [
      'title[0][value]' => 'Mon article de test',
      'body[0][value]'  => 'Contenu de l\'article',
    ],
    'Enregistrer'
  );

  $this->assertSession()->pageTextContains('Mon article de test');
  $this->assertSession()->addressMatches('/^\/node\/\d+$/');
}
```

---

## Assertions Disponibles

```php
// Status HTTP
$this->assertSession()->statusCodeEquals(200);
$this->assertSession()->statusCodeEquals(403);

// Contenu de la page
$this->assertSession()->pageTextContains('Texte attendu');
$this->assertSession()->pageTextNotContains('Texte absent');

// Éléments HTML
$this->assertSession()->elementExists('css', '.mon-element');
$this->assertSession()->elementNotExists('css', '.element-absent');
$this->assertSession()->elementTextContains('css', 'h1', 'Titre attendu');
$this->assertSession()->elementAttributeContains('css', 'a', 'href', '/chemin');

// XPath
$this->assertSession()->elementExists('xpath', '//table[@class="views-table"]');

// Liens
$this->assertSession()->linkExists('Texte du lien');
$this->assertSession()->linkByHrefExists('/chemin-du-lien');

// Champs de formulaire
$this->assertSession()->fieldExists('max_items');
$this->assertSession()->fieldValueEquals('max_items', '10');
$this->assertSession()->checkboxChecked('enable_feature');
$this->assertSession()->checkboxNotChecked('disable_feature');

// URL actuelle
$this->assertSession()->addressEquals('/chemin-attendu');
$this->assertSession()->addressMatches('/^\/node\/\d+$/');

// Headers de réponse
$this->assertSession()->responseHeaderEquals('Content-Type', 'application/json');

// Messages Drupal
$this->assertSession()->pageTextContains('Opération réussie');
```

---

## Tester un Endpoint REST ou JSON

```php
public function testEndpointJsonRetourneListe(): void {
  // Créer des données de test
  Node::create(['type' => 'article', 'title' => 'Article 1', 'status' => 1])->save();
  Node::create(['type' => 'article', 'title' => 'Article 2', 'status' => 1])->save();

  // Accéder à l'endpoint
  $this->drupalGet('/api/articles', ['query' => ['_format' => 'json']]);
  $this->assertSession()->statusCodeEquals(200);
  $this->assertSession()->responseHeaderEquals('Content-Type', 'application/json');

  // Décoder et vérifier la réponse
  $content = $this->getSession()->getPage()->getContent();
  $data = json_decode($content, TRUE);

  $this->assertIsArray($data);
  $this->assertCount(2, $data);
  $this->assertEquals('Article 1', $data[0]['title'][0]['value']);
}

public function testEndpointNecessiteAuthentification(): void {
  $this->drupalGet('/api/articles/private', ['query' => ['_format' => 'json']]);
  $this->assertSession()->statusCodeEquals(401);
}
```

---

## Helpers Utiles dans `BrowserTestBase`

```php
// Cliquer sur un lien
$this->clickLink('Modifier');

// Cliquer sur un bouton
$this->click('.mon-bouton');

// Trouver un élément et interagir
$page = $this->getSession()->getPage();
$element = $page->find('css', '.mon-selecteur');
$this->assertNotNull($element);
$element->click();

// Vérifier les classes CSS et attributs dans le HTML rendu
$this->assertSession()->elementExists('css', 'article.node--type-article');
$this->assertSession()->elementAttributeContains('css', 'img', 'loading', 'lazy');

// Récupérer le contenu HTML brut (pour vérifier les libraries chargées)
$html = $this->getSession()->getPage()->getContent();
$this->assertStringContainsString('mon_theme/css/global.css', $html);

// Récupérer le code de statut HTTP actuel
$code = $this->getSession()->getStatusCode();

// Recharger la page
$this->getSession()->reload();

// Récupérer un nœud créé lors du test
$node = $this->drupalCreateNode([
  'type'   => 'article',
  'title'  => 'Test node',
  'status' => 1,
]);

// Créer un fichier de test
$file = $this->createFile(['filename' => 'test.txt', 'uri' => 'public://test.txt']);
```

---

## Tester les Permissions par Rôle

```php
// ❌ D10- : /** @dataProvider rolesProvider */
// ✅ D11 / PHPUnit 11
#[\PHPUnit\Framework\Attributes\DataProvider('rolesProvider')]
public function testAccesSelonRole(array $permissions, int $expected_code): void {
  $user = empty($permissions)
    ? $this->drupalCreateUser()  // Utilisateur sans permission spéciale
    : $this->drupalCreateUser($permissions);

  $this->drupalLogin($user);
  $this->drupalGet('/admin/config/mon-module');
  $this->assertSession()->statusCodeEquals($expected_code);
}

public static function rolesProvider(): array {
  return [
    'sans permission'                    => [[], 403],
    'avec administer mon module'         => [['administer mon module'], 200],
    'avec access administration pages'   => [['access administration pages'], 403],
  ];
}
```
