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



