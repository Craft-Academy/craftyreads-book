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



[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/2fb2811ef6ce0a34d92751331a00cea610b6d3a0)

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



[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/63d29821bd316f698f64d63f27e87cbeb36a2c07)

L'étape naturelle d'après est donc de simplement faire passer le test :

`src/app.controller.ts`

```diff
 -11,7 +11,7 @@ export class AppController {
 <!DOCTYPE html>
 <html>
   <head>
-      <title>Page Title</title>
+      <title>Crafty Reads</title>
   </head>
   <body>
       <p>Hello, World!</p>
```

Un petit `npx playwright test` nous indique que le test est maintenant vert !

Bon. On n'a rien fait de particulièrement compliqué ici. Je rappelle encore une fois que l'objectif ce walking skeleton est de nous faire avancer dans la "configuration" de notre projet, de prendre un petit peu de temps au départ pour mettre en place les outils qui nous feront gagner un temps fou au fur et à mesure du développement.

Se pose donc maintenant la question suivante : j'ai expliqué précédement que la deuxième étape du walking skeleton était d'utiliser un vrai adapter primaire (ici l'UI), mais on garde pour l'instant un stub pour l'adapteur secondaire (la base de données).

Seulement voilà : nos tests se lancent depuis l'UI, donc depuis un "vrai" site qui tourne. Comment configurer notre "stub" dans ces conditions ? Notre test n'a pas accès au stub, le site tourne dans un processus complètement différent des tests !

Rappelons-nous le rôle le plus important du walking skeleton : avancer dans la configuration du projet et la mise en place de l'architecture logicielle sous-jacente et des différents outils.

Le rôle du walking skeleton n'est PAS d'avoir un code parfait tout de suite !

Il nous reste donc ici quelques éléments à gérer :

- l'injection de dépendances de NestJS
- l'utilisation de htmx

L'objectif de cette seconde étape de notre walking skeleton va donc être de mettre en place ces éléments. Le test depuis l'UI sera temporairement non-optimal, mais ils nous sera toutefois utile pour avancer.

Le plan sera donc le suivant :

- on va afficher un simple formulaire avec un seul champ "title" pour ajouter le titre du livre
- on va se servir de notre use case AddBookUseCase dans le controller pour nous forcer à configurer l'injection de dépendances de NestJS
- notre use case étant déjà testé unitairement avec le stub, on va pour l'instant simplement partir du principe que si le use case ne throw pas d'erreur, c'est que le livre a bien été enregistré
- on va renvoyer "book added" dans le controller lorsque le livre est ajouté.

Il manque évidemment ici dans le cadre d'un vrai test d'acceptation le fait de vérifier que le livre est réellement dans la base de données (il faudrait même le vérifier à travers l'api publique et non directement via la base de données). Comme pour l'instant nous n'avons pas la fonctionnalité de récupérer un livre dans l'api publique du projet, on se contente ici d'un test imparfait qui nous fait simplement avancer dans la configuration globale du projet.

A la fin du walking skeleton, ce test sera mis à jour et sera cette fois-çi beaucoup plus pertinent.

Let's go !



[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/847ac8e0ca2948594a9617007436c55c7a8fc980)

J'ai supprimé le test d'exemple et l'ai remplacé par le test qui nous intéresse :

`test/add-book.spec.ts`
```diff
 -0,0 +1,12 @@
+import { test, expect } from '@playwright/test';
+
+test.describe('Feature: Adding a book', () => {
+  test('Example: User can add a book', async ({ page }) => {
+    await page.goto('http://localhost:3000');
+
+    await page.getByLabel(/title/i).fill('Clean Code');
+    await page.getByText(/add book/i).click();
+
+    await expect(page.getByText(/book added/i)).toBeVisible();
+  });
+});
```

Rien de bien compliqué ici, le test parle de lui-même.

Venons-en maintenant à la modification du controller :

`src/app.controller.ts`
```diff
 -11,11 +11,15 @@ export class AppController {
 <!DOCTYPE html>
 <html>
   <head>
-      <title>Crafty Reads</title>
+    <title>Crafty Reads</title>
   </head>
   <body>
-      <p>Hello, World!</p>
-</body>
+    <form>
+      <label for="title">Title</label>
+      <input type="text" id="title" name="title">
+      <button type="submit">Add book</button>
+    </form>
+  </body>
 </html>`;
   }
 }
```

On-ne-peut plus simple là aussi ! On fait les choses rapidement, mais proprement. Dans le sens où on est couvert par notre test ;)

Bon évidemment, le test ne passe pas :

{% embed url="https://app.arcade.software/share/pJ3CCEiA8i6ra1kI8AND" %}



[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/3ddc7b664b69186c7dddda0d7814eac35122c0fd)

Avant de nous concentrer sur la partie configuration + htmx, concentrons-nous à faire passer ce test.

La façon la plus simple de le faire passer est tout simplement de vérifier l'existence d'une query dans l'url, ce qui implique que le formulaire a été envoyé, et afficher "Book added" dans ce cas :

`src/app.controller.ts`
```diff
 -6,7 +6,7 @@ export class AppController {
   constructor(private readonly appService: AppService) {}

   @Get()
-  getHello(): string {
+  getHello(@Query() query?: { title: string }): string {
     return `
 // ...
   <body>
     <form>
+    ${query?.title ? '<p>Book added</p>' : ''}
       <label for="title">Title</label>
       <input type="text" id="title" name="title">
       <button type="submit">Add book</button>
```

Le decorator @Query permet de récuperer les paramètres de query de l'url. Quand le formulaire est validé, la méthode par défaut est "GET", encore une fois on fait au plus simple ici. A terme ça ne sera pas un GET ;)

S'il y a une query dans l'url, on affiche donc "Book added". Et le test passe :

{% embed url="https://app.arcade.software/share/gLvsvN3nVOT07I5dDwqa" %}

{% hint style="warning" %}
"Mais rien n'empêche ici d'écrire n'importe quoi dans l'url en tant que query title directement et le test passera toujours ! On ne teste rien ici finalement 🤷🏼"
{% endhint %}

C'est en effet une bonne remarque. C'est parce que le but de ce "test" ici n'est à cette étape pas de tester le bon fonctionnement de notre app, mais de nous servir de guide pour compléter la "step 0" de notre projet !

Prenons un exemple :

`src/app.controller.ts`
```diff
 -1,9 +1,9 @@
 import { Controller, Get, Query } from '@nestjs/common';
-import { AppService } from './app.service';
+import { AddBookUseCase } from './add-book.usecase';

 @Controller()
 export class AppController {
-  constructor(private readonly appService: AppService) {}
+  constructor(private readonly addBookUseCase: AddBookUseCase) {}
```

J'ai simplement ajouté notre `AddBookUseCase` en tant que dépendance. Et notre test ne passe plus ! Eh oui, on a oublié de configurer notre injection de dépendances...

{% embed url="https://app.arcade.software/share/TSutqxXQsi3ZQ3aPwUB1" %}

{% hint style="warning" %}
"Ouais enfin merci mais j'ai pas besoin d'un test automatisé pour me dire que j'ai oublié de configuer mon injection de dépendances hein...Quand j'ai mon app qui tourne en `watch`, l'erreur serait apparue directement aussi, même plus rapidement qu'avec le test !"
{% endhint %}

Et c'est complètement vrai !

Encore une fois, je rappelle qu'ici le rôle de notre walking skeleton est d'avoir tout de configuré, ici le test nous indique effectivement que nous avons mal configuré l'application, mais c'est un "bonus" en l'occurrence.



[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/fd72eb59f16581877ee269bc0c1824fb38a20555)

Voici les quelques modifications à apporter pour configurer correctement NestJS :

`src/book-repository.port.ts`
```diff
 -1,3 +1,3 @@
-export interface BookRepository {
-  save(book: { title: string }): Promise<void>;
+export abstract class BookRepository {
+  abstract save(book: { title: string }): Promise<void>;
 }
```

Ici j'ai changé l'interface en abstract class. Les plus puristes d'entre nous vont crier à l'hérésie !

En effet, on déconseille généralement d'utiliser les abstract class en lieu et place des interfaces, pour éviter d'avoir des implémentations par défaut, et parce que beaucoup de languages ne peuvent pas étendre plusieurs abstract classes.

Ceci étant dit, en TypeScript il faut savoir deux choses :

- les interfaces n'ont pas d'existence réelle une fois compilée. On ne peut donc pas s'en servir directement comme token pour l'injection de dépendances
- les abstracts classes peuvent être "implémentée" plutôt qu'étendues. Oui oui, on peut faire `class Toto implements MonAbstractClass`

Il faut cependant resté rigoureux sur le fait de ne pas ajouter de comportement par défaut dans ces classes pour respecter leur contrat de "port".

Si l'on tient vraiment à utiliser des interfaces, on peut utiliser un token manuellement, en exportant par exemple dans le même fichier que l'interface un `Symbol` du nom de l'interface, exemple : `export const BookRepository = new Symbol('BookRepository')`.

Il faudrait alors utiliser ce symbole dans l'injection de dépendances de NestJS, et passer par un `useFactory` pour correctement configurer le provider.

C'est beaucoup plus verbeux, donc par pragmatisme je conseille d'utiliser plutôt une abstract class :)

Ne reste plus qu'à ajouter le decorator `@Injectable()` dans notre `AddBookUseCase` pour indiquer à NestJS qu'il doit voir ses dépendances injectées.

```diff
 -1,5 +1,7 @@
+import { Injectable } from '@nestjs/common';
 import { BookRepository } from './book-repository.port';

+@Injectable()
 export class AddBookUseCase {
   constructor(private readonly bookRepository: BookRepository) {}
```

{% hint style="warning" %}
"Oula oula, mais attends, le principe de l'architecture hexagonale, la clean archi, tout ça tout ça là, c'est pas justement de séparer le framework du coeur de métier ? Qu'est-ce que ce decorator propre à NestJS vient faire dans notre beau code censé être framework-agnostique !"
{% endhint %}

Alors oui, mais non.

Encore une fois il s'agit ici d'être pragmatique. Il est en effet très important de séparer la logique métier du la logique technique du framework ou autre. Mais ici il faut relativiser : ce n'est qu'une dépendance via un decorator. Decorator même pas pris en compte dans nos tests unitaires, donc complètement invisible pour nous !

Si l'on voulait se passer complètement de ce decorator, on pourrait passer par des `useFactory()` comme cité au dessus. Ce qui ferait écrire plus de code de configuration dans le framework.

Ici je décide donc d'utiliser les outils du framework pour me faciliter la vie, et dans l'éventualité où un jour je veuille chagner de framework Node (on sait très bien que ça n'arrivera jamais), j'aurais juste à retirer ces decoratos. Not a big deal ;)

Ne reste plus qu'à configurer notre module NestJS :

```diff
 -1,10 +1,18 @@
 import { Module } from '@nestjs/common';
 import { AppController } from './app.controller';
-import { AppService } from './app.service';
+import { AddBookUseCase } from './add-book.usecase';
+import { StubBookRepository } from './stub.book-repository';
+import { BookRepository } from './book-repository.port';

 @Module({
   imports: [],
   controllers: [AppController],
-  providers: [AppService],
+  providers: [
+    {
+      provide: BookRepository,
+      useValue: StubBookRepository,
+    },
+    AddBookUseCase,
+  ],
 })
 export class AppModule {}
```



[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/81821a1b885dff3739615613b38258e3e51d19fa)

Maintenant que la configuration du module est correcte, le test passe à nouveau :)

Je profite de cette étape pour faire un petit refactoring purement graphique en ajoutant une dose de tailwind :

`src/app.controller.ts`
```diff
export class AppController {
 <html>
   <head>
     <title>Crafty Reads</title>
+    <meta charset="UTF-8">
+    <meta name="viewport" content="width=device-width, initial-scale=1.0">
+    <script src="https://cdn.tailwindcss.com"></script>
   </head>
   <body>
-    <form>
-    ${query?.title ? '<p>Book added</p>' : ''}
-      <label for="title">Title</label>
-      <input type="text" id="title" name="title">
-      <button type="submit">Add book</button>
-    </form>
+    <main class="container mx-auto px-4 py-8">
+      ${query?.title ? '<p>Book added</p>' : ''}
+      <form class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
+        <div class="mb-4">
+          <label class="block text-gray-700 text-sm font-bold mb-2" for="title">
+            Title
+          </label>
+          <input class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline" id="title" type="text" name="title" placeholder="Enter book title" required>
+        </div>
+        <button class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline" type="submit">
+          Add Book
+        </button>
+      </form>
+    </main>
   </body>
 </html>`;
   }

```

L'objectif ce walking skeleton, je le rappelle, est de configurer toute la stack que l'on veut utiliser. Il reste notamment ici à tester htmx. Pour "l'installer", rien de plus simple :

`src/app.controller.ts`
```diff
 -15,6 +15,7 @@ export class AppController {
     <meta charset="UTF-8">
     <meta name="viewport" content="width=device-width, initial-scale=1.0">
     <script src="https://cdn.tailwindcss.com"></script>
+    <script src="https://unpkg.com/htmx.org@2.0.1"></script>
   </head>
   <body>
     <main class="container mx-auto px-4 py-8">
```

L'étape suivante est d'ajouter un nouveau test d'acceptation qui va vérifier que l'on ne peut pas ajouter un livre qui existe déjà en base de données.

Pour ce faire, nous allons tout simplement remplir deux fois le formulaire.

La première fois, dans la partie "arrange" du test, pour enregistrer le livre.

Puis une deuxiième fois dans la partie "act" du test, lorsque l'on veut tenter d'ajouter un livre dont le titre existe déjà.

Evidemment, tester si un livre existe déjà uniquement par son titre n'est pas suffisant en pratique ! Ici ça va cependant nous aider à tester plusieurs choses :

- vérifier que l'on utilise bien le use case (car pour l'instant on ne l'utilise pas)
- vérifier que htmx est capable de correctement remplacer le contenu du formulaire pour afficher l'erreur

Avec ça, il ne nous restera plus qu'à brancher un "vrai" adapter secondaire pour le BookRepository et notre walking skeleton sera terminé !

Voici le scénario ajouté dans les tests playwright :

`test/add-book.spec.ts`
```diff
+
+  test('Example: User cannot add a book that already exists', async ({
+    page,
+  }) => {
+    await page.goto('http://localhost:3000');
+    await page.getByLabel(/title/i).fill('Clean Code');
+    await page.getByText(/add book/i).click();
+
+    await page.getByLabel(/title/i).fill('Clean Code');
+    await page.getByText(/add book/i).click();
+
+    await expect(page.getByText(/book already exists/i)).toBeVisible();
+  });
 });
```

Ce test va évidemment échouer. On peut maintenant descendre d'un niveau et passer au test unitaire :

`src/add-book.spec.ts`
```diff
+  test('Example: User cannot add a book that already exists', async () => {
+    const bookRepository = new StubBookRepository();
+    bookRepository.booksByTitle.set('Clean Code', { title: 'Clean Code' });
+    const addBook = new AddBookUseCase(bookRepository);
+
+    const addingBook = addBook.execute({ title: 'Clean Code' });
+
+    await expect(addingBook).rejects.toThrowError(
+      new BookAlreadyExistsError('The book Clean Code already exists'),
+    );
+  });
 });
```

Plusieurs choses intéressantes ici :

- Tout d'abord, on ajoute manuellement le livre "Clean Code" dans notre StubRepository, qui devient au passage un Fake, car simulant une implémentation réelle.
- On s'attend ensuite à ce que le use case échoue avec une erreur de type `BookAlreadyExistsError`. Cette erreur n'existe évidemment pas encore.

Au passage, comme le StubBookRepository devient un Fake, on va changer le nom et l'appeler avec la nomenclature que j'utilise pour mes adapters classiques : TechnoXXXRepository. On est sur un repository en mémoire, donc partons pour le nom `InMemoryBookRepository`

`src/in-memory-book.repository.ts`
```diff
 import { BookRepository } from './book-repository.port';

 export class InMemoryBookRepository implements BookRepository {
+  booksByTitle = new Map<string, { title: string }>(); // J'en profite ici pour créer une Map référençant les livres par titre.
+
   lastSavedBook: { title: string } | undefined;

   async save(book: { title: string }): Promise<void> {
     this.lastSavedBook = book;
+    this.booksByTitle.set(book.title, book);
   }
 }
```



[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/1edb53d83106e46d670d50db2052832091adb94d)

Première chose à faire maintenant : créer l'erreur BookAlreadyExistsError pour que le test "compile" :

`book-already-exists.error.ts`
```diff
 -0,0 +1,5 @@
+export class BookAlreadyExistsError extends Error {
+  constructor(bookTitle: string) {
+    super(`The book ${bookTitle} already exists`);
+  }
+}
```

Le test échoue maintenant pour une première raison : on s'attend à ce qu'une erreur soit lancée, alors que pour l'instant aucune erreur n'est lancée.

La logique pour savoir qu'un livre existe va se positionner directement dans le repository. C'est en effet la solution la plus simple et la plus efficace que de déléguer ça au repository (et donc plus tard à la base de données sous-jacente) :

`src/book-repository.port.ts`
```diff
 -1,3 +1,4 @@
 export abstract class BookRepository {
   abstract save(book: { title: string }): Promise<void>;
+  abstract doesBookExist(title: string): Promise<boolean>;
 }
```

Notre implémentation in-memory est toute simple :

```diff
 -9,4 +9,8 @@ export class InMemoryBookRepository implements BookRepository {
     this.lastSavedBook = book;
     this.booksByTitle.set(book.title, book);
   }
+
+  async doesBookExist(title: string): Promise<boolean> {
+    return this.booksByTitle.has(title);
+  }
 }
```

Il nous faut maintenant faire une petite modification subtile de notre test pour avoir de nouveau la bonne raison de "fail" du test, en l'occurrence ici avoir par exemple un mauvais message dans l'erreur :

`src/add-book.spec.ts`
```diff
 -21,8 +22,6 @@ describe('Feature: Adding a book', () => {

     const addingBook = addBook.execute({ title: 'Clean Code' });

-    await expect(addingBook).rejects.toThrow(
-      new BookAlreadyExistsError('The book Clean Code already exists'),
-    );
+    await expect(addingBook).rejects.toThrow(new BookAlreadyExistsError('Foo'));
   });
 });
```

On peut maintenant implémenter la logique triviale :

`src/add-book.usecase.ts`
```diff
 -1,11 +1,15 @@
 import { Injectable } from '@nestjs/common';
 import { BookRepository } from './book-repository.port';
+import { BookAlreadyExistsError } from './book-already-exists.error';

 @Injectable()
 export class AddBookUseCase {
   constructor(private readonly bookRepository: BookRepository) {}

-  execute(book: { title: string }) {
+  async execute(book: { title: string }) {
+    if (await this.bookRepository.doesBookExist(book.title)) {
+      throw new BookAlreadyExistsError(book.title);
+    }
     return this.bookRepository.save(book);
   }
 }
```

Notre test fail maintenant pour la bonne raison :
```
Expected message: "The book Foo already exists"
Received message: "The book Clean Code already exists"
      at AddBookUseCase.execute (src/add-book.usecase.ts:38:27)
      at Object.<anonymous> (src/add-book.spec.ts:22:29)
```

Pourquoi est-ce si important d'avoir toujours un test qui fail pour la bonne raison avant de le faire passer ? Tout simplement pour éviter les faux positifs !

Un exemple typique : oublier d'ajouter une valeur à un enum. Ca marche dans le test car la valeur vaut `undefined` mais c'est évidemment pas ce qu'on attend et ça peut créer plein d'effets de bord !

Ne reste plus qu'à faire passer le test maintenant :

`src/add-book.spec.ts`
```diff
 -22,6 +22,8 @@ describe('Feature: Adding a book', () => {

     const addingBook = addBook.execute({ title: 'Clean Code' });

-    await expect(addingBook).rejects.toThrow(new BookAlreadyExistsError('Foo'));
+    await expect(addingBook).rejects.toThrow(
+      new BookAlreadyExistsError('Clean Code'),
+    );
   });
 });
```



[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/a44eab67457afa08f50b3b4566f4014b553722b3)

Rappelons-nous que l'un des objectifs restants du walking skeleton est de mettre en place htmx.

La prochaine étape va donc être de modifier quelque peu le HTML retourné par notre controller pour y inclure la magie htmx.

L'idée est toute simple :

- le formulaire va dorénavant utiliser la méthode POST
- on va "booster" le formulaire en y ajoutant l'attribut `hx-boost`
- htmx va donc effectuer une requête ajax pour récupérer soumettre le formulaire
- on va créer une nouvelle route dans notre controller pour répondre à cette soumission du formulaire et renvoyer le HTML correspondant
- htmx va remplacer notre HTML existant par celui renvoyé par notre API grâce aux attributs `hx-target` et `hx-swap`

Commençons par modifier la route affichant le formulaire :

`src/app.controller.ts`
```diff
 -6,7 +6,7 @@ export class AppController {
   constructor(private readonly addBookUseCase: AddBookUseCase) {}

   @Get()
-  getHello(@Query() query?: { title: string }): string {
+  index(): string { // plus besoin de récupérer la querystring, cette route devient simplement notre page de base
     return `
 <!DOCTYPE html>
 <html>
```

J'ajoute ensuite le script de htmx, et j'ajoute les attributs htmx vus plus haut :

`src/app.controller.ts`
```diff
 -15,12 +15,11 @@ export class AppController {
     <meta charset="UTF-8">
     <meta name="viewport" content="width=device-width, initial-scale=1.0">
     <script src="https://cdn.tailwindcss.com"></script>
+    <script src="https://unpkg.com/htmx.org@2.0.1" integrity="sha384-QWGpdj554B4ETpJJC9z+ZHJcA/i59TyjxEPXiiUgN2WmTyV5OEZWCD6gQhgkdpB/" crossorigin="anonymous"></script>
   </head>
   <body>
     <main class="container mx-auto px-4 py-8">
-      ${query?.title ? '<p>Book added</p>' : ''}
-      <form class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
+      <div id="add-book-form">
+        <form action="/" class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4" method="post" hx-boost="true" hx-target="#add-book-form" hx-swap="outerHTML">
           <div class="mb-4">
             <label class="block text-gray-700 text-sm font-bold mb-2" for="title">
               Title
```

Il faut ensuite ajouter la nouvelle route : celle qui va recevoir la soumission du formulaire, pour renvoyer soit le formulaire + le message de succces, soit le formulaire + le message d'erreur. Pour l'instant, on duplique comme des sales ! On nettoiera après, l'objectif est d'abord de faire passer les tests d'acceptation :

`src/app.controller.ts`
```diff
 -35,4 +34,56 @@ export class AppController {
   </body>
 </html>`;
   }
+
+  @Post()
+  async addBook(@Body() body: { title: string }) {
+    try {
+      await this.addBookUseCase.execute({ title: body.title });
+      return `
+        <div id="add-book-form">
+          <div class="bg-teal-100 border-t-4 border-teal-500 rounded-b text-teal-900 px-4 py-3 shadow-md" role="alert">
+            <div class="flex">
+              <div class="py-1"><svg class="fill-current h-6 w-6 text-teal-500 mr-4" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20"><path d="M2.93 17.07A10 10 0 1 1 17.07 2.93 10 10 0 0 1 2.93 17.07zm12.73-1.41A8 8 0 1 0 4.34 4.34a8 8 0 0 0 11.32 11.32zM9 11V9h2v6H9v-4zm0-6h2v2H9V5z"/></svg></div>
+              <div>
+                <p class="text-sm">Book added</p>
+              </div>
+            </div>
+          </div>
+          <form action="/" class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4" method="post" hx-boost="true" hx-target="#add-book-form" hx-swap="outerHTML">
+            <div class="mb-4">
+              <label class="block text-gray-700 text-sm font-bold mb-2" for="title">
+                Title
+              </label>
+              <input class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline" id="title" type="text" name="title" placeholder="Enter book title" required>
+            </div>
+            <button class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline" type="submit">
+              Add Book
+            </button>
+          </form>
+        </div>
+      `;
+    } catch (error) {
+      return `
+        <div id="add-book-form">
+          <div role="alert">
+            <div class="bg-red-500 text-white font-bold rounded-t px-4 py-2">
+              Book not added
+            </div>
+          </div>
+          <form action="/" class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4" method="post" hx-boost="true" hx-target="#add-book-form" hx-swap="outerHTML">
+            <div class="mb-4">
+              <label class="block text-gray-700 text-sm font-bold mb-2" for="title">
+                Title
+              </label>
+              <input class="shadow appearance-none border border-red-500 rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline" id="title" name="title" type="text" placeholder="Enter book title" name="title" placeholder="Enter book title" required>
+              <p class="text-red-500 text-xs italic mt-2">${(error as any).message}</p>
+            </div>
+            <button class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline" type="submit">
+              Add Book
+            </button>
+          </form>
+        </div>
+      `;
+    }
+  }
 }
```

Un point intéressant que les plus attentifs d'entre-vous auront probablement déjà relevé : je me suis trompé dans la configuration de mon module nest. En effet j'ai utilisé `useValue` au lieu de `useClass`. Aucun problème dans le typage car NestJS ne peut pas remonter l'information jusqu'à là où le `BookRepository` sera injecté. On est donc sur une belle erreur au runtime ! Erreur heureusement vite capturée grâce à notre test d'acceptation !

`src/app.module.ts`
```diff
 -10,7 +10,7 @@ import { BookRepository } from './book-repository.port';
   providers: [
     {
       provide: BookRepository,
-      useValue: InMemoryBookRepository,
+      useClass: InMemoryBookRepository,
     },
     AddBookUseCase,
   ],
```

Enfin, il nous faut modifier un petit peu le test d'acceptation.

En effet, actuellement j'utilise un repository in-memory. Lorsque l'on va lacner les tests d'acceptation, le serveur va se lancer, et le `InMemoryBookRepository` sera instancié. De fait, chaque test "utilise" de manière sous-jacente le même repository, ce qui rend les tests non isolés ! Les tests étant exécutés en parallèle, il est possible que le premier test fail directement car le livre Clean Code aura déjà été ajouté par un autre test.

C'est même encore pire : un même test peut ne pas être isolé avec lui-même, car si l'on exécute les tests playwright avec le mode `--ui`, il va exécuter les tests dans 3 navigateurs différents. Donc 6 tests ici en tout !

C'est un cas typique de gestion de "seed" de données dans un environnement de tests d'acceptation.

Il existe plein de façon de remédier à ça. Pour l'instant, on va y remédier de façon pas très propre. L'objectif est de nous faire avancer d'une étape en faisant passer les tests. Mais évidemment on reviendra dessus plus tard ;)

La solution la plus simple ici est juste de ne pas tenter d'insérer le même livre dans les 2 tests ! Ca assure l'isolation entre les deux tests. Quant à l'isolation du test avec lui-même, on peut tout simplement ajouter un nombre aléatoire en plus dans le nom du livre :

`test/add-book-spec.ts`
```diff
 -3,8 +3,9 @@ import { test, expect } from '@playwright/test';
 test.describe('Feature: Adding a book', () => {
   test('Example: User can add a book', async ({ page }) => {
     await page.goto('http://localhost:3000');
+    const rand = Math.floor(Math.random() * 1000000);

-    await page.getByLabel(/title/i).fill('Clean Code');
+    await page.getByLabel(/title/i).fill(`Clean Code ${rand}`);
     await page.getByText(/add book/i).click();

     await expect(page.getByText(/book added/i)).toBeVisible();
```

`test/add-book-spec.ts`
```diff
 -14,12 +15,14 @@ test.describe('Feature: Adding a book', () => {
     page,
   }) => {
     await page.goto('http://localhost:3000');
-    await page.getByLabel(/title/i).fill('Clean Code');
+    await page.getByLabel(/title/i).fill('The Pragmatic Programmer');
     await page.getByText(/add book/i).click();

-    await page.getByLabel(/title/i).fill('Clean Code');
+    await page.getByLabel(/title/i).fill('The Pragmatic Programmer');
     await page.getByText(/add book/i).click();

-    await expect(page.getByText(/book already exists/i)).toBeVisible();
+    await expect(
+      page.getByText(/the book the pragmatic programmer already exists/i),
+    ).toBeVisible();
   });
 });

```

Et le tour est joué ! Nos tests d'acceptation + nos tests unitaires passent :) C'est l'heure de faire un petit refactoring ;)



