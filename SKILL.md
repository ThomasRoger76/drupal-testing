---
name: drupal-testing
description: Use when writing PHPUnit tests for Drupal modules (Unit, Kernel, Functional, FunctionalJavascript), configuring phpunit.xml for Drupal, mocking services with UnitTestCase, testing Entity API or Config API with KernelTestBase, simulating user navigation with BrowserTestBase, testing AJAX with WebDriverTestBase, setting up CI/CD pipelines for Drupal automated testing in Drupal 8-11+, running PHPStan/phpcs/Rector static analysis on Drupal code, or testing an existing Drupal site without reinstalling using Drupal Test Traits (DTT)
---

# Drupal Testing — Référence Complète

## Overview

Référentiel complet des tests PHPUnit pour Drupal 8-11+ : infrastructure, 4 types de tests (Unit, Kernel, Functional, FunctionalJavascript), mocking, TDD, CI/CD.

## 🏗️ La Pyramide de Décision — Quel Test Utiliser ?

```
         ┌─────────────────────────┐
         │   FunctionalJavascript  │  ← AJAX, modales, drag-drop  (le plus lent)
         └─────────────────────────┘
        ┌───────────────────────────┐
        │       Functional          │  ← Navigation, users, formulaires
        └───────────────────────────┘
      ┌─────────────────────────────┐
      │          Kernel             │  ← Entity API, Config API, services, hooks
      └─────────────────────────────┘
    ┌───────────────────────────────┐
    │            Unit               │  ← Logique PHP pure, mocks  (le plus rapide)
    └───────────────────────────────┘
```

**Règle :** toujours choisir le niveau le plus bas (le plus rapide) qui couvre le besoin.

## Quick Decision Table

| Besoin | Type | Référence |
|--------|------|-----------|
| Tester une fonction PHP pure (calcul, validation, string) | Unit | [unit-tests.md](unit-tests.md) |
| Tester un service Drupal en isolation (avec mock) | Unit | [unit-tests.md](unit-tests.md) |
| Tester un algorithme avec plusieurs jeux de données | Unit + DataProvider | [unit-tests.md](unit-tests.md) |
| Tester l'Entity API (créer, charger, modifier un node) | Kernel | [kernel-tests.md](kernel-tests.md) |
| Tester la Config API ou un hook Drupal | Kernel | [kernel-tests.md](kernel-tests.md) |
| Tester un service qui nécessite un container Drupal réel | Kernel | [kernel-tests.md](kernel-tests.md) |
| Tester qu'une page répond HTTP 200/403 | Functional | [functional-tests.md](functional-tests.md) |
| Tester qu'un rôle a accès (ou non) à une page | Functional | [functional-tests.md](functional-tests.md) |
| Tester la soumission d'un formulaire | Functional | [functional-tests.md](functional-tests.md) |
| Tester un endpoint REST/JSON API | Functional | [functional-tests.md](functional-tests.md) |
| Tester un comportement AJAX | FunctionalJavascript | [javascript-tests.md](javascript-tests.md) |
| Tester une modale, un autocomplete, un drag-drop | FunctionalJavascript | [javascript-tests.md](javascript-tests.md) |
| Tester les classes CSS dans le HTML rendu | Functional — `elementExists('css', ...)` | [theme-layer-tests.md](theme-layer-tests.md) |
| Tester qu'une library CSS/JS est attachée | Functional — vérifier dans le HTML source | [theme-layer-tests.md](theme-layer-tests.md) |
| Tester les variables preprocess Twig | Kernel — appeler `preprocess_HOOK()` directement | [theme-layer-tests.md](theme-layer-tests.md) |
| Tester un render array complet (rendu HTML) | Kernel — `renderer->renderRoot()` | [theme-layer-tests.md](theme-layer-tests.md) |
| Tester qu'un email est envoyé | Kernel — `test_mail_collector` + `system.test_mail_collector` | [advanced-scenarios.md](advanced-scenarios.md) |
| Tester un QueueWorker | Kernel — `plugin.manager.queue_worker` | [advanced-scenarios.md](advanced-scenarios.md) |
| Tester le cron (`hook_cron`) | Kernel — `cron->run()` | [advanced-scenarios.md](advanced-scenarios.md) |
| Tester un EventSubscriber | Unit + mock du dispatcher | [advanced-scenarios.md](advanced-scenarios.md) |
| Tester un Field Formatter (HTML rendu) | Kernel — `renderer->renderRoot()` | [advanced-scenarios.md](advanced-scenarios.md) |
| Tester qu'une View affiche les bons résultats | Functional + Kernel | [advanced-scenarios.md](advanced-scenarios.md) |
| Tester un appel API externe | Unit/Kernel — GuzzleHttp `MockHandler` | [advanced-scenarios.md](advanced-scenarios.md) |
| Tester l'upload de fichier | FunctionalJavascript — `attachFileToField()` | [advanced-scenarios.md](advanced-scenarios.md) |
| Tester un nœud multilingue traduit | Kernel — `addTranslation()` | [advanced-scenarios.md](advanced-scenarios.md) |
| Tester une migration Drupal | Kernel — `MigrateTestBase` + `executeMigration()` | [advanced-scenarios.md](advanced-scenarios.md) |
| Tester un endpoint REST (POST/PATCH/DELETE) | Functional avec Basic Auth | [advanced-scenarios.md](advanced-scenarios.md) |
| **Tester un endpoint JSON:API** | **Functional — `BrowserTestBase` + assertions JSON** | **[advanced-scenarios.md](advanced-scenarios.md)** |
| Tester un `Drupal.behavior` JS en isolation | Jest — mock de `once` et `Drupal` global | [advanced-scenarios.md](advanced-scenarios.md) |
| Tests E2E JavaScript style Drupal core | Nightwatch.js | [advanced-scenarios.md](advanced-scenarios.md) |
| Tests BDD comportementaux (Gherkin) | Behat + DrupalExtension | [infrastructure.md](infrastructure.md) |
| Scénario Given/When/Then pour un formulaire | Behat + MinkExtension | [infrastructure.md](infrastructure.md) |
| Configurer PHPUnit pour Drupal | `phpunit.xml` | [infrastructure.md](infrastructure.md) |
| Lancer les tests avec Docker Compose | `docker compose exec php vendor/bin/phpunit` | [infrastructure.md](infrastructure.md) |
| Mettre en place le TDD | RED → GREEN → REFACTOR | [tdd-cicd.md](tdd-cicd.md) |
| Automatiser les tests en CI/CD | GitHub Actions / GitLab CI | [tdd-cicd.md](tdd-cicd.md) |
| **Pipeline GitLab CI complet (validate + test + coverage)** | **Stage validate→test→coverage avec seuil 70%** | **[tdd-cicd.md](tdd-cicd.md)** |
| **PHPStan niveau 6 pour analyse statique** | **`mglaman/phpstan-drupal` + rules Drupal** | **[static-analysis.md](static-analysis.md)** |
| **phpcs avec standard Drupal (2 espaces)** | **`drupal/coder` + `phpcs.xml` configuré** | **[static-analysis.md](static-analysis.md)** |
| **Rector pour corriger les dépréciations** | **`palantirnet/drupal-rector` — dry-run en CI** | **[static-analysis.md](static-analysis.md)** |
| **Scanner les dépréciations avant upgrade** | **`mglaman/drupal-check`** | **[static-analysis.md](static-analysis.md)** |
| **Tests sur site existant (pas de réinstall)** | **DTT `ExistingSiteBase` — 10× plus rapide** | **[drupal-test-traits.md](drupal-test-traits.md)** |
| **Smoke tests post-déploiement** | **DTT — teste la vraie DB, la vraie config** | **[drupal-test-traits.md](drupal-test-traits.md)** |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Raison |
|---------------------|------------------|--------|
| Functional test pour ce qu'un Kernel couvre | Choisir le niveau minimum suffisant | 10× plus lent sans bénéfice |
| `sleep(3)` dans les tests FunctionalJavascript | `assertSession()->waitForElement()` | Flaky tests — le timing varie |
| Ne pas mocker les services en Unit test | `$this->createMock(Interface::class)` | Le test ne teste plus une seule chose |
| `$this->assertTrue(TRUE)` | Assertions significatives | Test tautologique qui passe toujours |
| Tests qui dépendent de l'ordre d'exécution | Chaque test est autonome (`setUp/tearDown`) | Tests non déterministes |
| `$defaultTheme` non défini en Functional | Toujours déclarer `$defaultTheme = 'stark'` | Erreur à l'exécution |
| Tester les APIs Drupal core | Tester son propre code | Core est déjà testé |
| Charger tous les modules dans `$modules` | Charger uniquement ce qui est nécessaire | Tests lents, maintenance difficile |

## Évolution par Version Majeure

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| PHPUnit version | 6.x | 9.x | 10.x | **11.x** |
| `UnitTestCase` | ✅ | ✅ | ✅ | ✅ |
| `KernelTestBase` | ✅ | ✅ | ✅ | ✅ |
| `BrowserTestBase` | ✅ | ✅ | ✅ | ✅ |
| `WebDriverTestBase` | ✅ | ✅ | ✅ | ✅ |
| Annotations (`@group`, `@covers`) | ✅ | ✅ | ⚠️ deprecated | ❌ supprimées |
| Attributs PHP (`#[Group]`, `#[CoversClass]`) | ❌ | ❌ | ✅ partiel | ✅ **standard** |
| `@group` annotation | ✅ | ✅ | ✅ | ✅ |
| PHP Attributes pour annotations PHPUnit | ❌ | ❌ | ✅ partiel | ✅ |
| `drush test:run` | ❌ | ✅ | ✅ | ✅ |
| Nightwatch.js (JS tests core) | ✅ | ✅ | ✅ | ✅ |
| `createMock()` (PHPUnit) | ✅ | ✅ | ✅ | ✅ |
| `$modules` visibilité | `protected static` | `protected static` | `protected static` | `protected static` |
| `MigrateTestBase` (tests migration) | ✅ | ✅ | ✅ | ✅ |
| `test_mail_collector` (email testing) | ✅ | ✅ | ✅ | ✅ |
| `hook_deploy_N` dans les tests | ❌ | ✅ | ✅ | ✅ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Erreurs courantes et fixes. Ajouter après chaque debug de test.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions (v1.0 courante).

## See Also

- `drupal-core` — Entity API, Config API (ce que les Kernel tests testent)
- `drupal-config` — Config Management (tester la config via KernelTest)
- `drush` — `drush test:run`, commandes de lancement
- `docker-compose` — `docker compose exec php vendor/bin/phpunit`, ChromeDriver add-on
- `rector` — refactoring automatique PHP/Drupal (complémente [static-analysis.md](static-analysis.md))
- `xdebug` — step debug et couverture de code pour PHPUnit
