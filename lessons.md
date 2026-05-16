# Leçons — drupal-testing

Erreurs courantes et fixes découverts en usage réel. Mis à jour après chaque debug de test.

---

## Comment ajouter une leçon

Après chaque bug de test résolu :
1. Identifier si le skill aurait pu prévenir l'erreur
2. Ajouter une entrée avec symptôme + cause + correction + prévention
3. Ajouter une ligne dans `CHANGELOG.md`

---

## 2026-05-14 — Création du skill

### `$defaultTheme` non défini — Erreur au démarrage des Functional tests
- **Symptôme :** `RuntimeException: No theme found when the theme system is not initialized`
- **Cause :** `BrowserTestBase` et `WebDriverTestBase` requièrent `$defaultTheme` déclaré explicitement
- **Correct :** Ajouter `protected $defaultTheme = 'stark';` dans la classe de test
- **Prévention :** Toujours déclarer `$defaultTheme = 'stark'` (le thème minimal) dans tous les Functional tests

### `protected static $modules` oublié — Module non trouvé
- **Symptôme :** `The following module(s) are required: mon_module`
- **Cause :** Le module à tester n'est pas inclus dans `$modules`
- **Correct :** Ajouter le module dans `protected static $modules = ['mon_module']`
- **Prévention :** Inclure UNIQUEMENT les modules nécessaires (et leurs dépendances directes)

### `installEntitySchema('node')` oublié — Kernel test
- **Symptôme :** `PDOException: SQLSTATE[42S02]: Base table or view not found: 1146 Table 'node_field_data' doesn't exist`
- **Cause :** Le schéma de la table n'a pas été monté dans le `setUp()`
- **Correct :** Ajouter `$this->installEntitySchema('node')` dans `setUp()`
- **Prévention :** Pour chaque entité utilisée dans le test, ajouter le `installEntitySchema()` correspondant

### `installSchema('system', ['sequences'])` oublié — Auto-increment impossible
- **Symptôme :** `PDOException: SQLSTATE[42S02]: Table 'sequences' doesn't exist` ou ID toujours 0
- **Cause :** La table `sequences` gère les auto-increments Drupal — elle doit être installée
- **Correct :** Ajouter `$this->installSchema('system', ['sequences'])` dans `setUp()`
- **Prévention :** Ajouter systématiquement `sequences` quand on crée des entités en Kernel test

### `sleep()` dans les FunctionalJavascript — Tests flaky
- **Symptôme :** Test passe en local, échoue en CI (ou vice versa)
- **Cause :** `sleep()` utilise un timing fixe qui ne s'adapte pas à la vitesse machine
- **Correct :** Remplacer par `$this->assertSession()->waitForElement('css', '.selector', 10)`
- **Prévention :** Bannir tout `sleep()` dans les tests — utiliser les méthodes `waitFor*`

### `pageTextContains()` échoue sur un élément qui "existe" visuellement
- **Symptôme :** `assertSession()->pageTextContains('Mon texte')` échoue alors que le texte est visible dans le navigateur
- **Cause :** Le texte est dans un attribut `value`, `placeholder`, ou une balise `<title>` — pas dans le corps visible
- **Correct :** Utiliser `assertSession()->fieldValueEquals()` pour les champs ou `assertTitle()` pour le titre
- **Prévention :** `pageTextContains()` ne vérifie que le texte rendu dans le body visible — utiliser l'assertion spécifique selon le contexte

### Kernel test lent — Trop de modules dans `$modules`
- **Symptôme :** Tests Kernel qui prennent 30+ secondes quand ils devraient être en 2-5s
- **Cause :** Chargement de modules inutiles (`views`, `field_ui`, etc.) qui alourdissent le container
- **Correct :** N'inclure que les modules strictement nécessaires. Commencer avec le minimum et ajouter seulement si erreur
- **Prévention :** `$modules = ['mon_module']` puis ajouter selon les `requires` manquants

### Mocks dans `setUp()` puis redéfinition dans les méthodes de test — Conflits
- **Symptôme :** Certaines assertions ne fonctionnent pas car le mock a un comportement différent de celui attendu
- **Cause :** PHPUnit `createMock()` crée un mock strict — redéfinir `method()` après coup peut créer des conflits
- **Correct :** Configurer tous les `willReturn()` dans `setUp()` ou créer le mock localement dans chaque test qui en a besoin différemment
- **Prévention :** Si deux tests ont besoin de comportements différents → créer le mock dans chaque méthode de test, pas dans `setUp()`

### `assertWaitOnAjaxRequest()` timeout trop court en CI
- **Symptôme :** Tests FunctionalJavascript passent en local, timeout en CI/CD
- **Cause :** Machines CI plus lentes, ChromeDriver prend plus de temps
- **Correct :** Augmenter le timeout : `$this->assertSession()->assertWaitOnAjaxRequest(15000)` (15 secondes)
- **Prévention :** Utiliser des timeouts généreux en CI (10-15s) vs local (5s)

### `$this->config()->set()` dans KernelTestBase — `BadMethodCallException`
- **Symptôme :** `BadMethodCallException: Can not call set() on an immutable configuration object`
- **Cause :** `$this->config()` dans KernelTestBase retourne `ImmutableConfig` (lecture seule)
- **Correct :** `\Drupal::configFactory()->getEditable('nom.config')->set('key', $val)->save()`
- **Prévention :** `$this->config()` = lecture seule ; `configFactory()->getEditable()` = écriture

### `php -S` pour les tests Functional — Routes Drupal cassées
- **Symptôme :** Toutes les URLs retournent 404 sauf la page d'accueil
- **Cause :** Le serveur PHP built-in ne supporte pas `.htaccess`/`mod_rewrite`
- **Correct :** Utiliser Apache avec `mod_rewrite` activé
- **Prévention :** En CI, utiliser le service Apache ou nginx avec virtualhost configuré

### `find('css', ...)` sans null check — TypeError en FunctionalJavascript
- **Symptôme :** `PHP Fatal error: Call to a member function click() on null`
- **Cause :** `$page->find()` retourne `NodeElement|null` si l'élément n'existe pas encore
- **Correct :** `$el = $page->find('css', '...');\n$this->assertNotNull($el);\n$el->click();`
- **Prévention :** Toujours assertNotNull() après find() avant d'appeler une méthode dessus

### Email non capturé dans les tests — State vide
- **Symptôme :** `\Drupal::state()->get('system.test_mail_collector')` retourne `null`
- **Cause :** Le mail interface n'a pas été configuré avec `test_mail_collector` avant le test
- **Correct :** Ajouter dans setUp() : `\Drupal::configFactory()->getEditable('system.mail')->set('interface.default', 'test_mail_collector')->save()`
- **Prévention :** Toujours configurer le mail collector EN PREMIER dans setUp(), avant tout autre install

### `Node::create()` sans `uid` dans Kernel test — Erreur de contrainte
- **Symptôme :** `IntegrityConstraintViolationException: Field 'uid' doesn't have a default value`
- **Cause :** Les nœuds requièrent un auteur (`uid`) — sans DB user, ça échoue
- **Correct :** Toujours passer `'uid' => 0` (user anonyme) ou créer un user de test
- **Prévention :** Dans les Kernel tests, `'uid' => 0` est le raccourci pour les tests sans gestion d'utilisateurs


### 2026-05-14 — Module d'import/synchro API sans aucun test — Régression silencieuse
- **Symptôme :** Un changement dans la logique de validation (dates, statuts) casse silencieusement l'import — aucun feedback
- **Cause :** Logique de validation PHP pure dans des classes sans aucun Unit Test
- **Correct :** Les classes de validation et les DTOs sont parfaits pour les Unit Tests — aucune dépendance Drupal nécessaire
- **Prévention :** Toute classe avec des conditions `if/else` sur des données métier doit avoir des tests avec DataProvider couvrant : valeurs valides, invalides, nulles, cas limites
