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
{% embed url="https://gist.github.com/PCreations/149c66cc8e3063f32f5b2666c72d41cd" %}

Plusieurs petites choses à noter ici :

- Je ne me soucis pas de savoir où ranger le test, je le mets directement à la racine pour me faciliter la tache.
- Je ne me soucis pas non plus de configure Jest outre mesure, j'utilise la configuration par défaut de Nest. Pour l'instant ça me suffit amplement.
- Je tente de décrire le test d'un point de vue de l'acteur primaire, en l'occurence ici l'utilisateur.
- J'ai déjà dans ma tête la structure de base ports & adapters, je sais donc que je vais avoir besoin d'un adapter "BookRepository", que je vais avoir besoin d'un use case "AddBookUseCase" au niveau de la couche application.
- Je mets volontairement une mauvaise assertion dans le `expect`, pour que mon test échoue pour les bonnes raison. Pour l'instant le test échoue car `ReferenceError: BookRepository is not defined`

On va pouvoir maintenant implémenter les classes qui nous manquent pour que le test échoue pour les bonnes raisons :

{% embed url="https://gist.github.com/PCreations/2ab4df4b2dd9546823ab2b478eadf31f" %}

Cette fois on a bien le bon message d'erreur dans le test :

{% hint style="danger" %}
{% embed url="https://gist.github.com/PCreations/f2a916e478de991e0388ff699956f669" %}
{% endhint %}

Il ne reste plus qu'à faire passer le test en corrigeant l'assertion :

{% embed url="https://gist.github.com/PCreations/cb9e12cc96b40afa95f77d97cf41afcc" %}



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
{% embed url="https://gist.github.com/PCreations/e669f52e83cd04f4875ef052ab56eb42" %}

Le use case est maintenant dans son propre fichier.

`add-book.usecase.ts`
{% embed url="https://gist.github.com/PCreations/a043eda59e27417b1ac26a8fd11b82b7" %}

Un définit explicitement le contrat d'interface du port BookRepository :

`book-repository.port.ts`
{% embed url="https://gist.github.com/PCreations/f616c230ec1c2fa7e551bba1e3be92c0" %}

Il suffit maintenant d'implémenter un `StubBookRepository` (qui est en fait ici plutôt un spy, dans le sens où sa seule fonctionnalité pour le moment est "d'espionner" le fait qu'on a voulu sauvegarder un livre) :

`stub.book-repository.ts`
{% embed url="https://gist.github.com/PCreations/dc8d326fb322cef86b66a6eea3eef74f" %}



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
{% embed url="https://gist.github.com/PCreations/f24021a2c2e22648bd6747a9aa7b54bf" %}

`playwright.config.ts` : C'est la configuration générée lors de l'initialisation de playwright.
{% embed url="https://gist.github.com/PCreations/5fb78b56dd340cf7f6068d04b54fd669" %}

Ici ce qu'il est important de remarquer c'est que j'ai décommenté l'option webServer à la fin du fichier de configuration, pour que playwright lance automatiquement le serveur avant d'exécuter les tests. Pour l'instant c'est amplement suffisant pour ce que l'on veut tester :)

Maintenant vient la partie intéressante : notre premier test playwright :

`test/example.spec.ts`
{% embed url="https://gist.github.com/PCreations/76ea92183d701f895857b013fa8742e4" %}

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

{% embed url="https://gist.github.com/PCreations/96c51ec3ef5604e9550323d29cbafcd5" %}

Et cette fois on obtient bien le bon message d'erreur :

{% embed url="https://demo.arcade.software/N7mNTIRSddLsPjzkfFUp?embed&show_copy_link=true" %}



[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/63d29821bd316f698f64d63f27e87cbeb36a2c07)

L'étape naturelle d'après est donc de simplement faire passer le test :

`src/app.controller.ts`

{% embed url="https://gist.github.com/PCreations/c8b92dca97c73d2c28df1f5c95e15256" %}

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
{% embed url="https://gist.github.com/PCreations/a02214ff00b6aa928ab36f506cd1c186" %}

Rien de bien compliqué ici, le test parle de lui-même.

Venons-en maintenant à la modification du controller :

`src/app.controller.ts`
{% embed url="https://gist.github.com/PCreations/8d745e80c907e26db8cb5a2082652364" %}

On-ne-peut plus simple là aussi ! On fait les choses rapidement, mais proprement. Dans le sens où on est couvert par notre test ;)

Bon évidemment, le test ne passe pas :

{% embed url="https://app.arcade.software/share/pJ3CCEiA8i6ra1kI8AND" %}



[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/3ddc7b664b69186c7dddda0d7814eac35122c0fd)

Avant de nous concentrer sur la partie configuration + htmx, concentrons-nous à faire passer ce test.

La façon la plus simple de le faire passer est tout simplement de vérifier l'existence d'une query dans l'url, ce qui implique que le formulaire a été envoyé, et afficher "Book added" dans ce cas :

`src/app.controller.ts`
{% embed url="https://gist.github.com/PCreations/a8f666a88d68916cff6b22f3bc09f7f5" %}

Le decorator @Query permet de récuperer les paramètres de query de l'url. Quand le formulaire est validé, la méthode par défaut est "GET", encore une fois on fait au plus simple ici. A terme ça ne sera pas un GET ;)

S'il y a une query dans l'url, on affiche donc "Book added". Et le test passe :

{% embed url="https://app.arcade.software/share/gLvsvN3nVOT07I5dDwqa" %}

{% hint style="warning" %}
"Mais rien n'empêche ici d'écrire n'importe quoi dans l'url en tant que query title directement et le test passera toujours ! On ne teste rien ici finalement 🤷🏼"
{% endhint %}

C'est en effet une bonne remarque. C'est parce que le but de ce "test" ici n'est à cette étape pas de tester le bon fonctionnement de notre app, mais de nous servir de guide pour compléter la "step 0" de notre projet !

Prenons un exemple :

`src/app.controller.ts`
{% embed url="https://gist.github.com/PCreations/dccff5d48af03b980acf4d48d560643f" %}

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
{% embed url="https://gist.github.com/PCreations/6810931eba6f2472eabdb3dd30efb379" %}

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

{% embed url="https://gist.github.com/PCreations/bf14878853720f951309fd96c84837e7" %}

{% hint style="warning" %}
"Oula oula, mais attends, le principe de l'architecture hexagonale, la clean archi, tout ça tout ça là, c'est pas justement de séparer le framework du coeur de métier ? Qu'est-ce que ce decorator propre à NestJS vient faire dans notre beau code censé être framework-agnostique !"
{% endhint %}

Alors oui, mais non.

Encore une fois il s'agit ici d'être pragmatique. Il est en effet très important de séparer la logique métier du la logique technique du framework ou autre. Mais ici il faut relativiser : ce n'est qu'une dépendance via un decorator. Decorator même pas pris en compte dans nos tests unitaires, donc complètement invisible pour nous !

Si l'on voulait se passer complètement de ce decorator, on pourrait passer par des `useFactory()` comme cité au dessus. Ce qui ferait écrire plus de code de configuration dans le framework.

Ici je décide donc d'utiliser les outils du framework pour me faciliter la vie, et dans l'éventualité où un jour je veuille chagner de framework Node (on sait très bien que ça n'arrivera jamais), j'aurais juste à retirer ces decoratos. Not a big deal ;)

Ne reste plus qu'à configurer notre module NestJS :

{% embed url="https://gist.github.com/PCreations/eff81af95c9424d3476547ef56a5364d" %}



[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/81821a1b885dff3739615613b38258e3e51d19fa)

Maintenant que la configuration du module est correcte, le test passe à nouveau :)

Je profite de cette étape pour faire un petit refactoring purement graphique en ajoutant une dose de tailwind :

`src/app.controller.ts`
{% embed url="https://gist.github.com/PCreations/760e335dd6e8b2bb522d8263d9662d15" %}

L'objectif ce walking skeleton, je le rappelle, est de configurer toute la stack que l'on veut utiliser. Il reste notamment ici à tester htmx. Pour "l'installer", rien de plus simple :

`src/app.controller.ts`
{% embed url="https://gist.github.com/PCreations/95df6fa6e0482bb7db9e7db02c502312" %}

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
{% embed url="https://gist.github.com/PCreations/39b54500036f3d5ffb6217361a770e84" %}

Ce test va évidemment échouer. On peut maintenant descendre d'un niveau et passer au test unitaire :

`src/add-book.spec.ts`
{% embed url="https://gist.github.com/PCreations/87a00e74930e46569645c6c0c180fa45" %}

Plusieurs choses intéressantes ici :

- Tout d'abord, on ajoute manuellement le livre "Clean Code" dans notre StubRepository, qui devient au passage un Fake, car simulant une implémentation réelle.
- On s'attend ensuite à ce que le use case échoue avec une erreur de type `BookAlreadyExistsError`. Cette erreur n'existe évidemment pas encore.

Au passage, comme le StubBookRepository devient un Fake, on va changer le nom et l'appeler avec la nomenclature que j'utilise pour mes adapters classiques : TechnoXXXRepository. On est sur un repository en mémoire, donc partons pour le nom `InMemoryBookRepository`

`src/in-memory-book.repository.ts`
{% embed url="https://gist.github.com/PCreations/eaea7f124bbe19b67f446b3bf6b4b9b7" %}



[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/1edb53d83106e46d670d50db2052832091adb94d)

Première chose à faire maintenant : créer l'erreur BookAlreadyExistsError pour que le test "compile" :

`book-already-exists.error.ts`
{% embed url="https://gist.github.com/PCreations/504ba36ae222777b392e9e7e45e89173" %}

Le test échoue maintenant pour une première raison : on s'attend à ce qu'une erreur soit lancée, alors que pour l'instant aucune erreur n'est lancée.

La logique pour savoir qu'un livre existe va se positionner directement dans le repository. C'est en effet la solution la plus simple et la plus efficace que de déléguer ça au repository (et donc plus tard à la base de données sous-jacente) :

`src/book-repository.port.ts`
{% embed url="https://gist.github.com/PCreations/d0caba97c056c0400a933de56b1f6d22" %}

Notre implémentation in-memory est toute simple :

{% embed url="https://gist.github.com/PCreations/e1c944f7750aa9cdc3a1e503484364f4" %}

Il nous faut maintenant faire une petite modification subtile de notre test pour avoir de nouveau la bonne raison de "fail" du test, en l'occurrence ici avoir par exemple un mauvais message dans l'erreur :

`src/add-book.spec.ts`
{% embed url="https://gist.github.com/PCreations/7b7aaf0152f803c5d5b323212d438ebe" %}

On peut maintenant implémenter la logique triviale :

`src/add-book.usecase.ts`
{% embed url="https://gist.github.com/PCreations/d37d8789a5ca0673e4361fa98ba00e02" %}

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
{% embed url="https://gist.github.com/PCreations/af4b0a428f269594c293ae236ab9f959" %}



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
{% embed url="https://gist.github.com/PCreations/49a9a47bc4819b87b95c89ffbb7e6712" %}

J'ajoute ensuite le script de htmx, et j'ajoute les attributs htmx vus plus haut :

`src/app.controller.ts`
{% embed url="https://gist.github.com/PCreations/eb8af7594b7fe28191e41eb11f31674a" %}

Il faut ensuite ajouter la nouvelle route : celle qui va recevoir la soumission du formulaire, pour renvoyer soit le formulaire + le message de succces, soit le formulaire + le message d'erreur. Pour l'instant, on duplique comme des sales ! On nettoiera après, l'objectif est d'abord de faire passer les tests d'acceptation :

`src/app.controller.ts`
{% embed url="https://gist.github.com/PCreations/faf7174f4846e7f18fe9e33875848a25" %}

Un point intéressant que les plus attentifs d'entre-vous auront probablement déjà relevé : je me suis trompé dans la configuration de mon module nest. En effet j'ai utilisé `useValue` au lieu de `useClass`. Aucun problème dans le typage car NestJS ne peut pas remonter l'information jusqu'à là où le `BookRepository` sera injecté. On est donc sur une belle erreur au runtime ! Erreur heureusement vite capturée grâce à notre test d'acceptation !

`src/app.module.ts`
{% embed url="https://gist.github.com/PCreations/a8adce811a065ecbedbd1c783172e559" %}

Voici le résultat :

{% embed url="https://app.arcade.software/share/FoI67XeiQV0eRsTD37OT" %}

Enfin, il nous faut modifier un petit peu le test d'acceptation.

En effet, actuellement j'utilise un repository in-memory. Lorsque l'on va lacner les tests d'acceptation, le serveur va se lancer, et le `InMemoryBookRepository` sera instancié. De fait, chaque test "utilise" de manière sous-jacente le même repository, ce qui rend les tests non isolés ! Les tests étant exécutés en parallèle, il est possible que le premier test fail directement car le livre Clean Code aura déjà été ajouté par un autre test.

C'est même encore pire : un même test peut ne pas être isolé avec lui-même, car si l'on exécute les tests playwright avec le mode `--ui`, il va exécuter les tests dans 3 navigateurs différents. Donc 6 tests ici en tout !

C'est un cas typique de gestion de "seed" de données dans un environnement de tests d'acceptation.

Il existe plein de façon de remédier à ça. Pour l'instant, on va y remédier de façon pas très propre. L'objectif est de nous faire avancer d'une étape en faisant passer les tests. Mais évidemment on reviendra dessus plus tard ;)

La solution la plus simple ici est juste de ne pas tenter d'insérer le même livre dans les 2 tests ! Ca assure l'isolation entre les deux tests. Quant à l'isolation du test avec lui-même, on peut tout simplement ajouter un nombre aléatoire en plus dans le nom du livre :

`test/add-book-spec.ts`
{% embed url="https://gist.github.com/PCreations/d38bca921ade81b047c8debe88722483" %}

`test/add-book-spec.ts`
{% embed url="https://gist.github.com/PCreations/87c246be5c08d6c5a4a9ed35a2a13c13" %}

Et le tour est joué ! Nos tests d'acceptation + nos tests unitaires passent :) C'est l'heure de faire un petit refactoring ;)



[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/7ad7b6ee855f279b46ec590ab85bdc0d150d7459)

Avant de poursuivre dans l'implémentation de la première vraie fonctionnalité, on va en profiter pour faire un petit refactoring.

Actuellement, on retourne une string HTML directement depuis le controller.

Ca marche bien, mais à terme ce ne sera pas très pratique...

L'idéal serait de pouvoir utiliser des composants, un peu comme avec react !

Pour ce faire, on va utiliser typed-html, qui est petite librairie qui permet d'utiliser du jsx :)

### Installation et configuration de Storybook et typed-html

Pour installer storybook, rien de plus simple, il suffit de suivre le wizard :

`npx storybook@latest init`

On sélectionne ensuite le mode "html", et j'ai choisi "vite" comme bundler.

Par défaut, storybook installe aussi l'extension "addon-essentials", qui contient notamment la possibilité d'écrire de la documentation avec mdx.

Ce package introduit une dépendance à react, et notamment au package `@types/react`, qui va poser problème avec le typage JSX de typed-html. On va voir juste après comme remédier à ce problème.

Installons maintenant typed-html :

`npm i typed-html`

Cette petite librairie permet d'écrire du JSX sans utiliser react, et convertir ce jsx en chaîne de caractères. Pour ce faire, il simplement configurer le fichier `tsconfig.json` :

{% embed url="https://gist.github.com/PCreations/7f89318f1066109dcd8225ec4f25b82a" %}

`elements` est l'objet utilisé par typed-html pour générer la string HTML à partir du JSX.

À l'heure où j'écris ces lignes, il y a un léger bug dans le `package.json` de `typed-html`, une erreur sur la destination esm des fichiers.

Tant que la PR n'est pas mergée, on peut simplement créer un patch avec `patch-package` :

`npm i -D patch-package`
`mkdir patches`
`cd patches && vi typed-html+3.0.1.patch`

Le contenu du patch :

```patch
diff --git a/node_modules/typed-html/dist/esm/src/elements.d.ts b/node_modules/typed-html/dist/esm/elements.d.ts
similarity index 68%
rename from node_modules/typed-html/dist/esm/src/elements.d.ts
rename to node_modules/typed-html/dist/esm/elements.d.ts
index 4d68df8..7d6e280 100644
--- a/node_modules/typed-html/dist/esm/src/elements.d.ts
+++ b/node_modules/typed-html/dist/esm/elements.d.ts
@@ -1,6 +1,6 @@
-/// <reference path="../../../src/jsx/element-types.d.ts" />
-/// <reference path="../../../src/jsx/events.d.ts" />
-/// <reference path="../../../src/jsx/intrinsic-elements.d.ts" />
+/// <reference path="../../src/jsx/element-types.d.ts" />
+/// <reference path="../../src/jsx/events.d.ts" />
+/// <reference path="../../src/jsx/intrinsic-elements.d.ts" />
 declare type AttributeValue = number | string | Date | boolean | string[];
 export interface Children {
     children?: AttributeValue;
diff --git a/node_modules/typed-html/dist/esm/src/elements.d.ts.map b/node_modules/typed-html/dist/esm/elements.d.ts.map
similarity index 100%
rename from node_modules/typed-html/dist/esm/src/elements.d.ts.map
rename to node_modules/typed-html/dist/esm/elements.d.ts.map
diff --git a/node_modules/typed-html/dist/esm/src/elements.js b/node_modules/typed-html/dist/esm/elements.js
similarity index 100%
rename from node_modules/typed-html/dist/esm/src/elements.js
rename to node_modules/typed-html/dist/esm/elements.js
diff --git a/node_modules/typed-html/dist/esm/src/elements.js.map b/node_modules/typed-html/dist/esm/elements.js.map
similarity index 100%
rename from node_modules/typed-html/dist/esm/src/elements.js.map
rename to node_modules/typed-html/dist/esm/elements.js.map
diff --git a/node_modules/typed-html/dist/esm/src/tsconfig.tsbuildinfo b/node_modules/typed-html/dist/esm/tsconfig.tsbuildinfo
similarity index 100%
rename from node_modules/typed-html/dist/esm/src/tsconfig.tsbuildinfo
rename to node_modules/typed-html/dist/esm/tsconfig.tsbuildinfo
```

Enfin il suffit d'ajouter un script `postinstall` dans notre `package.json` et le tour est joué. On en profite aussi pour supprimer le package `@types/react` de nos node_modules pour ne pas avoir d'interference avec le typage de JSX :

{% embed url="https://gist.github.com/PCreations/b20228f97a5f7ea1326725dd64502249" %}

Il nous reste une petite configuration pour storybook et vite à mettre en place pour que le bundler vite utilise correctement `elements.createElement`.

Il suffit d'ajouter un fichier `vite.config.ts` dans le dossier `.storybook` :

`.storybook/vite.config.ts`
```ts
import { defineConfig } from 'vite'

export default defineConfig({
  esbuild: {
    jsxFactory: 'elements.createElement',
  }
})
```

Et évidemment préciser dans la configuration de Storybook qu'on veut utiliser ce fichier de configuration :

`.storybook/main.ts`
```ts
import type { StorybookConfig } from '@storybook/html-vite';

const config: StorybookConfig = {
  stories: ['../src/**/*.mdx', '../src/**/*.stories.@(js|jsx|mjs|ts|tsx)'],
  addons: [
    '@storybook/addon-links',
    '@chromatic-com/storybook',
    '@storybook/addon-interactions',
    '@storybook/addon-essentials'
  ],
  framework: {
    name: '@storybook/html-vite',
    options: {
      builder: {
        viteConfigPath: './.storybook/vite.config.ts',
      }
    },
  },
};
export default config;
```

Dernière étape, simplement ajouter les scripts de tailwind et de htmx dans le `head` des pages storybook. Rien de plus simple encore une fois, il suffit de créer un fichier `preview-head.html` dans le dossier `.storybook` :

`.storybook/preview-head.html`
```html
<script src="https://cdn.tailwindcss.com"></script>
<script src="https://unpkg.com/htmx.org@2.0.1" integrity="sha384-QWGpdj554B4ETpJJC9z+ZHJcA/i59TyjxEPXiiUgN2WmTyV5OEZWCD6gQhgkdpB/" crossorigin="anonymous"></script>
```



[Commit checkpoint](https://github.com/Craft-Academy/craftyreads/commit/345e1d453d42f5cbcdc227bfe5d5c6488fc49d8e)

Maintenant que storybook est configuré, commençons par le premier composant : le bouton.

`src/components/button.tsx`
```ts
import * as elements from 'typed-html';

export const Button = ({ name, type }: { name: string; type: string }) => {
  return (
    <button
      class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline"
      type={type}
    >
      {name}
    </button>
  );
};
```

Et la story qui va avec :

`src/stories/Button.stories.ts`
```ts
import type { StoryObj, Meta } from '@storybook/html';
import { Button } from '../components/button';

// More on how to set up stories at: https://storybook.js.org/docs/writing-stories#default-export
const meta = {
  title: 'Example/Button',
  tags: ['autodocs'],
  render: (args) => {
    // You can either use a function to create DOM elements or use a plain html string!
    // return `<div>${label}</div>`;
    return Button(args);
  },
  argTypes: {
    name: { control: 'color' },
    type: { control: 'text' },
  },
} satisfies Meta<{ name: string; type: string }>;

export default meta;
type Story = StoryObj<{ name: string; type: string }>;

// More on writing stories with args: https://storybook.js.org/docs/writing-stories/args
export const Primary: Story = {
  args: {
    name: 'Foo Bar',
    type: 'submit',
  },
};
```

Ne reste plus qu'à démarrer storybook :

`npm run storybook`

Et voici le résultat :

{% embed url="https://app.arcade.software/share/wC02h1eM5D63sTi1Ep8T" %}

Créons maintenant le layout de l'application :

`src/components/layout.tsx`
```ts
import * as elements from 'typed-html';

export const Layout = ({ children }: elements.Attributes) => {
  return (
    <html>
      <head>
        <title>Crafty Reads</title>
        <meta charset="UTF-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
        <script src="https://cdn.tailwindcss.com"></script>
        <script
          src="https://unpkg.com/htmx.org@2.0.1"
          integrity="sha384-QWGpdj554B4ETpJJC9z+ZHJcA/i59TyjxEPXiiUgN2WmTyV5OEZWCD6gQhgkdpB/"
          crossorigin="anonymous"
        ></script>
        <script src="//unpkg.com/alpinejs" defer=""></script>
      </head>
      <body>
        <main class="container mx-auto px-4 py-8">{children}</main>
        {/* Cette partie concerne la gestion des messages type toast. J'utilise des attributs Alpine.js pour gérer la transition et l'affichage du toast.
        On va voir ensuite comment gérer le toast dans notre code.
        */}
        <div
          aria-live="assertive"
          class="pointer-events-none fixed inset-0 flex items-end px-4 py-6 sm:items-start sm:p-6"
        >
          <div class="flex w-full flex-col items-center space-y-4 sm:items-end">
            <div
              id="toast-container"
              x-transition:enter="transform ease-out duration-300 transition"
              x-transition:enter-start="translate-y-2 opacity-0 sm:translate-y-0 sm:translate-x-2"
              x-transition:enter-end="translate-y-0 opacity-100 sm:translate-x-0"
              x-transition:leave="transition ease-in duration-100"
              x-transition:leave-start="opacity-100"
              x-transition:leave-end="opacity-0"
            ></div>
          </div>
        </div>
      </body>
    </html>
  );
};
```

Passons maintenant au composant du formulaire :

`src/components/add-book-form.tsx`
```ts
import * as elements from 'typed-html';
import { Button } from './button';
import { Toast } from './toast';

export const AddBookForm = ({
  inputPlaceholder,
  toast,
}: {
  inputPlaceholder: string;
  toast?: {
    type: 'error' | 'success';
    title: string;
    message: string;
  };
}) => {
  return (
    <div id="add-book-form">
      <form
        action="/"
        class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4"
        method="post"
        hx-boost="true"
        hx-target="#add-book-form"
        hx-swap="outerHTML"
      >
        <div class="mb-4">
          <label class="block text-gray-700 text-sm font-bold mb-2" for="title">
            Title
          </label>
          <input
            class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
            id="title"
            type="text"
            name="title"
            placeholder={inputPlaceholder}
            required="required"
          />
        </div>
        <Button name="Add Book" type="submit" />
      </form>
      {toast && (
        <Toast title={toast.title} message={toast.message} type={toast.type} />
      )}
    </div>
  );
};
```

Rien que de très classique ici, on a un formulaire qui envoie une requête à l'API et qui affiche un toast en fonction du résultat, le tout en utilisant htmx pour éviter de recharger la page grâce à l'attribut hx-boost.

Voici comment est géré le toast :

`src/components/toast.tsx`
```ts
import * as elements from 'typed-html';

export const Toast = ({
  title,
  message,
  type,
}: {
  title: string;
  message: string;
  type: 'error' | 'success';
}) => {
  return (
    <div
      id="toast-container"
      hx-swap-oob="true"
      x-data="{ open: true }"
      x-init="setTimeout(() => open = false, 1500)"
      class="pointer-events-auto w-full max-w-sm overflow-hidden rounded-lg bg-white shadow-lg ring-1 ring-black ring-opacity-5"
      x-show="open"
    >
      <div class="p-4">
        <div class="flex items-start">
          <div class="flex-shrink-0">
            {`<svg
              class="h-6 w-6 text-${type === 'success' ? 'green' : 'red'}-400"
              fill="none"
              viewBox="0 0 24 24"
              stroke-width="1.5"
              stroke="currentColor"
              aria-hidden="true"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M9 12.75L11.25 15 15 9.75M21 12a9 9 0 11-18 0 9 9 0 0118 0z"
              />
            </svg>`}
          </div>
          <div class="ml-3 w-0 flex-1 pt-0.5">
            <p class="text-sm font-medium text-gray-900">{title}</p>
            <p class="mt-1 text-sm text-gray-500">{message}</p>
          </div>
          <div class="ml-4 flex flex-shrink-0">
            <button
              type="button"
              x-on:click="open = false"
              class="inline-flex rounded-md bg-white text-gray-400 hover:text-gray-500 focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:ring-offset-2"
            >
              <span class="sr-only">Close</span>
              {`<svg
                class="h-5 w-5"
                viewBox="0 0 20 20"
                fill="currentColor"
                aria-hidden="true"
              >
                <path d="M6.28 5.22a.75.75 0 00-1.06 1.06L8.94 10l-3.72 3.72a.75.75 0 101.06 1.06L10 11.06l3.72 3.72a.75.75 0 101.06-1.06L11.06 10l3.72-3.72a.75.75 0 00-1.06-1.06L10 8.94 6.28 5.22z" />
              </svg>`}
            </button>
          </div>
        </div>
      </div>
    </div>
  );
};
```

Plusieurs choses à noter :

- On utilise x-data="{ open: true }" pour gérer l'ouverture et la fermeture du toast.
- On utilise x-init="setTimeout(() => open = false, 1500)" pour fermer le toast après 1.5s.
- On utilise hx-swap-oob="true" qui permet d'injecter le toast en dehors du formulaire, dans le container définit par le même `id` dans le body.
- typed-html ne gère pas les svg natif, on doit donc utiliser une string pour les insérer.

Ne reste plus qu'à modifier le controller pour utiliser ces composants :

`src/app.controller.tsx`
```ts
import * as elements from 'typed-html';
import { Body, Controller, Get, Post } from '@nestjs/common';
import { AddBookUseCase } from './add-book.usecase';
import { Layout } from './components/layout';
import { AddBookForm } from './components/add-book-form';

@Controller()
export class AppController {
  constructor(private readonly addBookUseCase: AddBookUseCase) {}

  @Get()
  index(): string {
    return (
      <Layout>
        <AddBookForm inputPlaceholder="Enter book title" />
      </Layout>
    );
  }

  @Post()
  async addBook(@Body() body: { title: string }) {
    try {
      await this.addBookUseCase.execute({ title: body.title });
      return (
        <AddBookForm
          inputPlaceholder="Enter book title"
          toast={{
            type: 'success',
            title: 'Book added',
            message: 'The book has been added to the list',
          }}
        />
      );
    } catch (error) {
      return (
        <AddBookForm
          inputPlaceholder="Enter book title"
          toast={{
            type: 'error',
            title: 'Book not added',
            message: (error as any).message,
          }}
        />
      );
    }
  }
}
```

Attention ici j'ai renommé le controller en .tsx car on utilise du jsx à travers les composants. On verra plus tard s'il est souhaitable de faire autrement ;)

Et voici le résultat :

{% embed url="https://app.arcade.software/share/AdrW62uQqkmSYYfBzPC3" %}



