# FunctionalJavascript Tests — `WebDriverTestBase`

## Quand Utiliser

- Comportements AJAX (rechargement partiel de page)
- Autocomplete, modales, dropdowns dynamiques
- Drag-and-drop (tableaux de poids Drupal, médias)
- Interactions qui nécessitent JavaScript pour fonctionner
- Durée : **30-120 secondes** — le plus lent, nécessite ChromeDriver

**Règle :** utiliser UNIQUEMENT pour les interactions qui ne peuvent pas être testées avec `BrowserTestBase`. Si le comportement fonctionne sans JS → utiliser `BrowserTestBase`.

---

## Prérequis — ChromeDriver avec DDEV

```bash
# Installer Selenium Standalone Chrome
ddev get ddev/ddev-selenium-standalone-chrome
ddev restart

# Vérifier que le service tourne
ddev describe | grep selenium
```

```xml
<!-- phpunit.xml — configuration ChromeDriver -->
<env name="MINK_DRIVER_ARGS_WEBDRIVER" value='["chrome", {"browserName":"chrome","goog:chromeOptions":{"args":["--disable-gpu","--headless","--no-sandbox","--disable-dev-shm-usage","--window-size=1280,1024"]}}, "http://selenium:4444/wd/hub"]'/>
<env name="SIMPLETEST_BASE_URL" value="http://web"/>
```

---

## Structure de Base

```php
<?php
// web/modules/custom/mon_module/tests/src/FunctionalJavascript/MonAjaxTest.php
namespace Drupal\Tests\mon_module\FunctionalJavascript;

use Drupal\FunctionalJavascriptTests\WebDriverTestBase;

/**
 * @group mon_module
 */
final class MonAjaxTest extends WebDriverTestBase {

  protected static $modules = ['mon_module', 'node'];
  protected $defaultTheme = 'stark';  // OBLIGATOIRE

  protected function setUp(): void {
    parent::setUp();
    // Configuration identique aux Functional tests
  }
}
```

---

## Attentes AJAX — Éviter les Flaky Tests

```php
public function testFormulaireAjaxMiseAJour(): void {
  $admin = $this->drupalCreateUser(['administer mon module']);
  $this->drupalLogin($admin);

  $this->drupalGet('/mon-module/ajax-form');

  // Déclencher un changement qui lance une requête AJAX
  $this->getSession()->getPage()->selectFieldOption('categorie', 'article');

  // ✅ Attendre que l'élément AJAX apparaisse (timeout = 10 secondes)
  $this->assertSession()->waitForElement('css', '#sous-categories-wrapper select', 10);

  // ✅ Attendre du texte spécifique
  $this->assertSession()->waitForText('Sous-catégories disponibles', 10);

  // ✅ Attendre la fin de TOUTES les requêtes AJAX Drupal
  $this->assertSession()->assertWaitOnAjaxRequest();

  // Maintenant vérifier le résultat
  $this->assertSession()->elementExists('css', '#sous-categories-wrapper select');
  $this->assertSession()->pageTextContains('Sous-catégorie A');
}

// ❌ À ne jamais faire
public function testAvecSleepFlaky(): void {
  $this->click('#bouton-ajax');
  sleep(3);  // ← Flaky : trop court sur machine lente, trop long en CI
  $this->assertSession()->pageTextContains('Résultat');
}
```

---

## Méthodes d'Attente Disponibles

```php
// Attendre qu'un élément CSS existe (timeout en secondes)
$element = $this->assertSession()->waitForElement('css', '.ajax-result', 10);

// Attendre qu'un texte apparaisse
$this->assertSession()->waitForText('Chargement terminé', 10);

// Attendre qu'un élément disparaisse
$this->assertSession()->waitForElementRemoved('css', '.loading-spinner', 10);

// Attendre la fin des requêtes AJAX Drupal
$this->assertSession()->assertWaitOnAjaxRequest(10000);  // millisecondes

// Attendre qu'un élément avec texte spécifique soit présent
$this->assertSession()->waitForElementVisible('css', '.modal', 10);

// Attendre qu'un champ soit visible
$this->assertSession()->waitForField('sous_categorie', 10);
```

---

## Interactions JavaScript

```php
public function testClicBouton(): void {
  $page = $this->getSession()->getPage();

  // find() retourne NodeElement|null — toujours vérifier avant d'appeler une méthode
  $button = $page->find('css', '#mon-bouton');
  $this->assertNotNull($button, 'Le bouton #mon-bouton doit être présent dans la page');
  $button->click();
  $this->assertSession()->assertWaitOnAjaxRequest();
}

public function testRemplirAutocomplete(): void {
  $this->drupalGet('/node/add/article');
  $page = $this->getSession()->getPage();

  // Taper dans un champ autocomplete
  $page->fillField('field_tags[0][target_id]', 'Dru');

  // Attendre les suggestions autocomplete
  $this->assertSession()->waitForElement('css', '.ui-autocomplete', 10);

  // Sélectionner la première suggestion
  $page->find('css', '.ui-autocomplete li:first-child')->click();

  // Vérifier que la sélection a fonctionné
  $this->assertSession()->waitForField('field_tags[0][target_id]');
}

public function testModalOuvreEtFerme(): void {
  $page = $this->getSession()->getPage();

  // Ouvrir la modale
  $page->find('css', '.open-modal-button')->click();
  $modal = $this->assertSession()->waitForElement('css', '.ui-dialog', 10);
  $this->assertNotNull($modal);

  // Vérifier le contenu
  $this->assertSession()->pageTextContains('Titre de la modale');

  // Fermer la modale
  $page->find('css', '.ui-dialog-titlebar-close')->click();
  $this->assertSession()->waitForElementRemoved('css', '.ui-dialog', 10);
}

public function testDragAndDrop(): void {
  $admin = $this->drupalCreateUser(['administer blocks']);
  $this->drupalLogin($admin);

  $this->drupalGet('/admin/structure/block');
  $page = $this->getSession()->getPage();

  // Drag and drop (requiert le driver webdriver)
  $draggable = $page->find('css', '.draggable:first-child');
  $target    = $page->find('css', '.draggable:last-child');

  $draggable->dragTo($target);
  $this->assertSession()->assertWaitOnAjaxRequest();
}
```

---

## Exécuter du JavaScript

```php
public function testAvecJavaScriptExecute(): void {
  // Exécuter du JS directement dans la page
  $result = $this->getSession()->evaluateScript('return document.title');
  $this->assertStringContainsString('Mon Module', $result);

  // Déclencher un événement JS
  $this->getSession()->executeScript(
    "document.querySelector('.mon-element').dispatchEvent(new Event('change'))"
  );
  $this->assertSession()->assertWaitOnAjaxRequest();

  // Scroll vers un élément
  $this->getSession()->executeScript(
    "document.querySelector('#element-en-bas').scrollIntoView()"
  );
}
```

---

## Screenshots pour le Debugging

```php
public function testAvecScreenshot(): void {
  $this->drupalGet('/ma-page');

  // Prendre un screenshot si le test échoue
  try {
    $this->assertSession()->pageTextContains('Texte attendu');
  } catch (\Exception $e) {
    // Screenshot automatique
    $this->createScreenshot('/tmp/echec-test-' . time() . '.png');
    throw $e;
  }
}

protected function onNotSuccessfulTest(\Throwable $t): void {
  // Screenshot automatique sur tout échec
  $this->createScreenshot('/tmp/test-failure-' . static::class . '-' . time() . '.png');
  parent::onNotSuccessfulTest($t);
}
```

---

## Différences Clés vs `BrowserTestBase`

| | `BrowserTestBase` | `WebDriverTestBase` |
|--|------------------|---------------------|
| Moteur | Client HTTP simulé (Goutte) | ChromeDriver réel |
| JavaScript | ❌ Non exécuté | ✅ Exécuté |
| AJAX | ❌ | ✅ |
| Vitesse | Rapide (~5-15s) | Lent (~30-120s) |
| ChromeDriver requis | ❌ | ✅ |
| Screenshots | ❌ | ✅ |
| `waitForElement()` | ❌ | ✅ |

---

## Troubleshooting FunctionalJavascript

| Erreur | Cause | Solution |
|--------|-------|---------|
| `Connection refused to ChromeDriver` | Selenium non démarré | `ddev restart` après installation de selenium |
| Test flaky (passe/échoue aléatoirement) | `sleep()` ou timing non contrôlé | Remplacer par `waitForElement()` |
| `Element not found` immédiatement | Pas d'attente après action AJAX | `assertWaitOnAjaxRequest()` après chaque action |
| Screenshot vide | URL de base incorrecte | Vérifier `SIMPLETEST_BASE_URL` dans phpunit.xml |
| Timeout sur waitForElement | Element n'arrive jamais | Vérifier la console JS (erreurs JS ?) |
