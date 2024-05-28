# Full page
[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/fc812b3b3cec8ea318ec9a2f0e465e3e425b68aa)
# Installation du projet

Pour démarrer le plus simplement possible le projet, j'utilise la commande de la cli de Nest, en strict mode :
`npx @nestjs/cli@latest new craftyreads --strict`

J'utilise npm, et le projet est prêt !

Il y a évidemment plusieurs fichiers scaffoldés pour nous, on fera le ménage plus tard ;)


[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/22ff006f15cc0075147455808afeedcf24b6525f)
# Walking Skeleton

Une notion très importante lorsque l'on commence un nouveau projet est de commencer par un Walking Skeleton. Cette approche permet de mettre en place de bout en bout une première fonctionnalité, très très petite, mais qui le mérite de mettre en place l'architecture logicielle et infra dès le début du projet, permettant aux deux d'évoluer en parallèle.

L'idée n'est pas ici de graver dans le marbre un choix de technologie, mais plutôt de faire naitre la structure Ports & Adapters.

Pour ce faire c'est très simple, il suffit de trouver 2 acteurs :

- Un acteur primaire, qui va "driver" l'application
- Un acteur secondaire, qui va être "drivé" par l'application

Je prends ici l'exemple simpliste de l'ajout d'un livre. Comme on peut le voir je ne me soucis pour l'instant pas du tout des détails. Un livre = un titre. Je sais évidemment que je ne sera pas la représentation finale d'un livre, mais aussi l'objectif est de pouvoir mettre en production très rapidement quelque chose.

Attention, quand je dis mettre en "production", je ne parle pas ici de mettre à disposition des utilisateurs l'outil. La production reste "cachée" au tout début. L'objectif est principalement de savoir que l'on peut déployer rapidement, pour lancer la machine. On verra dans un second temps comment mettre en place une pipeline de déploiement minimale. L'idée est de s'accorder un peu de temps au début du projet pour mettre en place ces éléments plus techniques, pour pouvoir avancer extrêmement rapidement, de manière fluide, après.

## Les 4 étapes du Walking Skeleton

L'objectif du Walking Skeleton est d'avoir très rapidement un workflow branché depuis l'acteur primaire jusqu'à l'acteur secondaire. L'acteur primaire ici c'est l'utilisateur, qui à travers son navigateur va pouvoir ajouter un livre. L'acteur secondaire c'est la base de données, dans laquelle va être sauvegardé ce livre. Qui dit acteur dit "adapters" dans l'architecture hexagonale. L'objectif du walking skeleton est donc de faire émerger ces adapters à partir de la création de "ports" représentant les "contrats" nécessaires au workflow.

La première étape consiste à utiliser les tests comme premier adapter primaire, le test runner va donc jouer le rôle du premier acteur primaire, en lieu et place de l'utilisateur (puisqu'il n'y a pas d'UI encore). On branchera donc un stub pour les tests.
La seconde étape consiste cette fois à utiliser un vrai adapter primaire, en l'occurence une UI dans notre cas, pour que l'utilisateur puisse maintenant jouer le rôle d'acteur primaire, et déclencher lui-même à travers un formulaire l'ajout d'un livre. Au passage ça nous permettra de tester toute la partie branchement avec HTMX :)
La dernière étape : brancher une vraie base de données. Si l'on ne souhaite pas choisir tout de suite une base de données, on peut opter pour un fake in-memory qui reproduit le rôle basique d'une db, mais ça doit être fonctionnel de bout en bout :)

# Etape 1

Comme convenu, on commence par un test :

`src/add-book.spec.ts`
```diff
 -0,0 +1,12 @@
+describe('Feature: Adding a book', () => {
+  test('Example: User can add a book', async () => {
+    const bookRepository = new BookRepository();
+    const addBook = new AddBookUseCase();
+
+    await addBook.execute({ title: 'Clean Code' });
+
+    expect(bookRepository.lastSavedBook).toEqual({
+      title: 'Clean OSdod',
+    });
+  });
+});

```

Plusieurs petites choses à noter ici :

- Je ne me soucis pas de savoir où ranger le test, je le mets directement à la racine pour me faciliter la tache.
- Je ne me soucis pas non plus de configure Jest outre mesure, j'utilise la configuration par défaut de Nest. Pour l'instant ça me suffit amplement.
- Je tente de décrire le test d'un point de vue de l'acteur primaire, en l'occurence ici l'utilisateur.
- J'ai déjà dans ma tête la structure de base ports & adapters, je sais donc que je vais avoir besoin d'un adapter "BookRepository", que je vais avoir besoin d'un use case "AddBookUseCase" au niveau de la couche application.
- Je mets volontairement une mauvaise assertion dans le `expect`, pour que mon test échoue pour les bonnes raison. Pour l'instant le test échoue car `ReferenceError: BookRepository is not defined`

On va pouvoir maintenant implémenter les classes qui nous manquent pour que le test échoue pour les bonnes raisons :

```diff
 -1,7 +1,18 @@
 describe('Feature: Adding a book', () => {
   test('Example: User can add a book', async () => {
     // Pour l'instant je me contente d'écrire le code le plus simple possible
+    class BookRepository {
+      lastSavedBook: { title: string } | undefined;
+    }
+
     const bookRepository = new BookRepository();
-    const addBook = new AddBookUseCase(bookRepository);
+
+    class AddBookUseCase {
+      async execute(book: { title: string }) {
+        bookRepository.lastSavedBook = book;
+      }
+    }
+
+    const addBook = new AddBookUseCase();

     await addBook.execute({ title: 'Clean Code' });
```

Cette fois on a bien le bon message d'erreur dans le test :

{% hint style="danger" %}
```diff
- Expected  - 1
+ Received  + 1

  Object {
-   "title": "Clean OSdod",
+   "title": "Clean Code",
  }
```
{% endhint %}

Il ne reste plus qu'à faire passer le test en corrigeant l'assertion :

```diff
 -17,7 +17,7 @@ describe('Feature: Adding a book', () => {
     await addBook.execute({ title: 'Clean Code' });

     expect(bookRepository.lastSavedBook).toEqual({
-      title: 'Clean OSdod',
+      title: 'Clean Code',
     });
   });
 });

```


[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/462bd624b8559f1f0b0be18b587e2d1007dacc61)
Maintenant que notre test passe, il faut passer à l'étape 2 du walking skeletong : utiliser l'UI comme primary adapter.

Le problème que l'on a ici c'est que tout est déclaré directement dans le test. Il exite des dépendances implicites.

En effet, la classe `AdddBookUseCase` dépend directement de `bookRepository` que l'on déclare dans le test.

Ca marche dans notre exemple, mais ça ne marchera plus dès que l'on voudra changer d'adapter : passer des tests à l'UI par exemple.

C'est ici que rentre en jeu la notion de "ports" dans l'architecture ports & adapters.

Le principe est très simple : l'application (ici composée d'un seul et unique use case : `AddBookUseCase`) doit définir les contrats qu'elle comprend en entrée, et avec lesquels elle peut communiquer en sortie.

Dans notre cas, il y a deux contrats :

- le contrat lié à l'actor primaire, qui va "driver" l'application. Ici il s'agit d'expliciter le fait qu'il existe un point d'entrée "AddBookuseCase" dans l'application. Dans le pattern de base, il faudrait ici créer une interface explicitement, qui serait implémentée par le use case. En pratique, on peut souvent s'en passer. Ce n'est donc pas tout à fait de l'architecture ports & adapters "by the books".
- le contrat lié à la communication avec le monde extérieur. Ici il s'agit de pouvoir sauvegarder un livre. L'application définit donc qu'elle est capable de communiquer avec n'importe qui qui implementerait le contrat `save(book: { title: string })`. Et comment on représente un contrat généralement ? Via une interface !

Pour instaurer la mise en place de ces contrats, et surtout du deuxième donc (puisque le premier peut être omis si on décide de s'éloigner de la vision "pure"), il faut que notre couche application, le use case AddBookUseCase déclare qu'il peut communiquer avec le contrat lié à la communication extérieure. Plutôt que de dépendre directement de l'implémentation concrète BookRepository, il dépend maintenant de l'interface BookRepository. L'implémentation concrète se voit injectée dans le constructeur. Ce faisant, AddBookUseCase déclare qu'il doit nécessairement pouvoir communiquer avec un objet qui respecte le contrat de l'interface BookRepository. Il s'en fiche de savoir l'implémentation concrète derrière.

`add-book.spec.ts`
```diff
+import { AddBookUseCase } from './add-book.usecase';
+import { StubBookRepository } from './stub.book-repository';
+
 describe('Feature: Adding a book', () => {
   test('Example: User can add a book', async () => {
-    class BookRepository {
-      lastSavedBook: { title: string } | undefined;
-    }
-
-    const bookRepository = new BookRepository();
-
-    class AddBookUseCase {
-      async execute(book: { title: string }) {
-        bookRepository.lastSavedBook = book;
-      }
-    }
+    const bookRepository = new StubBookRepository(); // ici on instantie notre stub de BookRepository

-    const addBook = new AddBookUseCase();
+    const addBook = new AddBookUseCase(bookRepository); // la dépendance est maintenant injectée.

     await addBook.execute({ title: 'Clean Code' });
```

Le use case est maintenant dans son propre fichier.

`add-book.usecase.ts`
```diff
 -0,0 +1,9 @@
+import { BookRepository } from './book-repository.port';
+
+export class AddBookUseCase {
+  constructor(private readonly bookRepository: BookRepository) {}
+
+  execute(book: { title: string }) {
+    return this.bookRepository.save(book);
+  }
+}
```

Un définit explicitement le contrat d'interface du port BookRepository :

`book-repository.port.ts`
```diff
+export interface BookRepository {
+  save(book: { title: string }): Promise<void>;
+}
```

Il suffit maintenant d'implémenter un `StubBookRepository` (qui est en fait ici plutôt un spy, dans le sens où sa seule fonctionnalité pour le moment est "d'espionner" le fait qu'on a voulu sauvegarder un livre) :

`stub.book-repository.ts`
```diff
 -0,0 +1,9 @@
+import { BookRepository } from './book-repository.port';
+
+export class StubBookRepository implements BookRepository {
+  lastSavedBook: { title: string } | undefined;
+
+  async save(book: { title: string }): Promise<void> {
+    this.lastSavedBook = book;
+  }
+}
```


[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/bd8ff7031fd545ac1b286ba8dc6f4467d9e9395e)
# Etape 2 : branchement de l'UI

Pour cette deuxième étape de notre walking skeleton, on veut que l'utilisateur devienne l'acteur primaire, et donc qu'il dirige l'application à travers une UI.

Le but d'un walking skeleton est aussi de configurer ce qui a trait aux tests, ici on va configurer Playwright pour s'en servir pour tester l'UI.

Je ne dis pas que l'on va écrire tous nos tests d'acceptation sous forme de test Playwright, mais on verra plus tard que ceux-ci pourront nous sauver la mise ;)

Pour l'instant l'objectif est de réduire les zones d'ombres du projet concernant notamment HTMX (que je n'ai encore jamais utilisé), et la configuration globale. Cette étape nous permet donc de tester tout ça pas à pas !

## Installation de Playwright

Première chose à faire : installer et initialiser playwright :

`❯ npm init playwright@latest`

Voici comment je l'ai configuré :

```console
Need to install the following packages:
create-playwright@1.17.132
Ok to proceed? (y) y
(##################) ⠹ reify:create-playwright: http fetch GET 200 https://registry.npmjs.org/create-playwright/-/create-play
Getting started with writing end-to-end tests with Playwright:
Initializing project in '.'
✔ Where to put your end-to-end tests? · test
✔ Add a GitHub Actions workflow? (y/N) · false
✔ Install Playwright browsers (can be done manually via 'npx playwright install')? (Y/n) · true
Installing Playwright Test (npm install --save-dev @playwright/test)…
```

Comme on a déjà un dossier test à la racine, autant utiliser ce dossier !

On se retrouve donc avec ces changements :

`package.json`
```diff
       "ts"
     ],
     "rootDir": "src",
-    "testRegex": ".*\.spec\.ts$",
+    "testRegex": "src/.*\.spec\.ts$", // Ici je modifie la configuration Jest pour qu'il ne cherche pas à exécuter les tests présents dans le dossier test, mais uniquement ceux dans le dossier src/
     "transform": {
       "^.+\.(t|j)s$": "ts-jest"
     },
```

`playwright.config.ts` : C'est la configuration générée lors de l'initialisation de playwright.
```diff
 -0,0 +1,77 @@
+import { defineConfig, devices } from '@playwright/test';
+
+/**
+ * Read environment variables from file.
+ * https://github.com/motdotla/dotenv
+ */
+// require('dotenv').config();
+
+/**
+ * See https://playwright.dev/docs/test-configuration.
+ */
+export default defineConfig({
+  testDir: './test',
+  /* Run tests in files in parallel */
+  fullyParallel: true,
+  /* Fail the build on CI if you accidentally left test.only in the source code. */
+  forbidOnly: !!process.env.CI,
+  /* Retry on CI only */
+  retries: process.env.CI ? 2 : 0,
+  /* Opt out of parallel tests on CI. */
+  workers: process.env.CI ? 1 : undefined,
+  /* Reporter to use. See https://playwright.dev/docs/test-reporters */
+  reporter: 'html',
+  /* Shared settings for all the projects below. See https://playwright.dev/docs/api/class-testoptions. */
+  use: {
+    /* Base URL to use in actions like `await page.goto('/')`. */
+    // baseURL: 'http://127.0.0.1:3000',
+
+    /* Collect trace when retrying the failed test. See https://playwright.dev/docs/trace-viewer */
+    trace: 'on-first-retry',
+  },
+
+  /* Configure projects for major browsers */
+  projects: [
+    {
+      name: 'chromium',
+      use: { ...devices['Desktop Chrome'] },
+    },
+
+    {
+      name: 'firefox',
+      use: { ...devices['Desktop Firefox'] },
+    },
+
+    {
+      name: 'webkit',
+      use: { ...devices['Desktop Safari'] },
+    },
+
+    /* Test against mobile viewports. */
+    // {
+    //   name: 'Mobile Chrome',
+    //   use: { ...devices['Pixel 5'] },
+    // },
+    // {
+    //   name: 'Mobile Safari',
+    //   use: { ...devices['iPhone 12'] },
+    // },
+
+    /* Test against branded browsers. */
+    // {
+    //   name: 'Microsoft Edge',
+    //   use: { ...devices['Desktop Edge'], channel: 'msedge' },
+    // },
+    // {
+    //   name: 'Google Chrome',
+    //   use: { ...devices['Desktop Chrome'], channel: 'chrome' },
+    // },
+  ],
+
+  /* Run your local dev server before starting the tests */
+  webServer: {
+    command: 'npm run start',
+    url: 'http://127.0.0.1:3000',
+    reuseExistingServer: !process.env.CI,
+  },
+});
```

Ici ce qu'il est important de remarquer c'est que j'ai décommenté l'option webServer à la fin du fichier de configuration, pour que playwright lance automatiquement le serveur avant d'exécuter les tests. Pour l'instant c'est amplement suffisant pour ce que l'on veut tester :)

Maintenant vient la partie intéressante : notre premier test playwright :

`test/example.spec.ts`
```diff
 -0,0 +1,8 @@
+import { test, expect } from '@playwright/test';
+
+test('has title', async ({ page }) => {
+  await page.goto('http://localhost:3000');
+
+  // Expect a title "to contain" a substring.
+  await expect(page).toHaveTitle(/Crafty Reads/);
+});
```

Comme on peut le voir, c'est un test absolument trivial (j'ai même pas pris la peine de changer le nom du fichier). L'idée est ici d'avoir la plus petite étape possible intéressante pour avancer dans notre découverte de HTMX, et dans la configuration global du projet.

Pour l'instant, on n'a évidemment pas encore de page HTML avec comme titre "Crafty Reads", on s'attend donc à ce que les tests fail:

`npx playwright test`

Par défaut, cette commande exécute les tests en mode headless. A la fin des tests, l'output apparaît dans la console, mais aussi dans une page HTML dressant le rapport d'exécution :

L'embed ci-dessous est interactive :)

{% embed url="https://demo.arcade.software/gRMCYbfJPmJ5OBGPAe4B?embed&show_copy_link=true" %}


[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/7e4f77977fc0e0805d28640cc61e2c339439f53b)
La prochaine étape est donc de faire échouer notre test playwright mais pour la bonne raison.

C'est-à-dire que l'on s'attend à recevoir un message d'erreur indiquant que le titre n'est pas le bon.

J'ai donc demandé à mon pote ChatGPT de me générer le html minimum, il a fait un peu de zèle, mais voilà ce que j'ai donc modifié :

```diff
 -7,6 +7,15 @@ export class AppController {

   @Get()
   getHello(): string {
-    return this.appService.getHello();
+    return `
+<!DOCTYPE html>
+<html>
+  <head>
+      <title>Page Title</title> // Je mets ici volontairement le mauvais titre !
+  </head>
+  <body>
+      <p>Hello, World!</p> // Bon ça c'est la ChatGPT Touch, c'était pas utile.
+</body>
+</html>`;
   }
 }
```

Et cette fois on obtient bien le bon message d'erreur :

{% embed url="https://demo.arcade.software/N7mNTIRSddLsPjzkfFUp?embed&show_copy_link=true" %}


