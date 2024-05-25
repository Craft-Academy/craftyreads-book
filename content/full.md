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


