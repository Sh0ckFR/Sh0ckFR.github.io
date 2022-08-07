+++
author = "Sh0ckFR"
title = "Régis et le Game Hacking"
date = "2022-08-03"
description = "Régis et le Game Hacking"
tags = [
    "gamehacking",
]
+++

Bonjour tout le monde, c'est Régis cette fois :)

![Sh0ckFR Game Hacking Regis](/images/regis.gif)

Cela vous ait sûrement déjà arrivé dans les jeux vidéos en ligne, tomber sur un joueur plus fort que vous, en étant persuadé que celui-ci trichait.

Voir même avoir des retours de vos amis en jeu comme par exemple `il triche pas c'est toi le noob tu sais pas viser t'as 0 skill`.

De quoi casser de belles amitiés liées depuis des années en quelques minutes, cela dit comme dirait [Antoine Daniel, la batte de baseball permet également de créer des liens sociaux](https://www.youtube.com/watch?v=YGl6Y79cB6c).

Je vous propose tout de même de voir ici si Régis est effectivement un débutant ou si les joueurs en face ont de grandes probabilités de tricher en ligne.

Attention tout de même, cet article n'a pas du tout pour vocation à détruire l'expérience des autres joueurs en ligne, je considère que ceux qui me liront jusqu'au bout auront une certaine maturité même si on va aller plutôt loin en abordant le cas de GTA5, `il s'agit ici d'un article de recherche avant tout` et n'a aucune vocation à promouvoir ce genre de pratiques, et dans tous les cas vous n'aurez pas de codes tout prêts, il s'agira seulement d'une bonne introduction pour débutants.

## Comment un cheat (logiciel de triche) fonctionne t'il ?

Un cheat est un programme développé comme tout autre programme, en différents languages de programmation, que cela soit du `C, du C++, du C#, du Java, du Python, du Rust, du Brainfuck (ouais non on va s'arrêter là)`.

Bref vous l'aurez compris, on peut écrire notre programme de triche dans tous les languages possibles mais concrêtement comment ça fonctionne ?

Si vous avez déjà vu quelques cours en C durant vos études, vous saurez que des `pointers` sont présents dans la mémoire d'un programme, pour les autres qui n'ont pas eu la chance de suivre les cours barbants de comment faire un jeu vidéo en C ou recoder la libc, les pointers sont en fait des emplacements présents dans la mémoire qui représentent l'adressse en mémoire de toutes sortes de variables (`integer` qui sont des chiffres ronds, `float` qui sont des chiffres à virgules, `boolean` qui sont des conditions `vraies` ou `fausses` etc...

Donc on imagine notre programme tourner (notre jeu en question), et on se représente la mémoire de celui-ci qui est une suite d'octects (bytes, 1 byte soit 1 octect = 8 bits)

Maintenant imaginons que nous souhaitons modifier ces valeurs en mémoire pour modifier par exemple le pointer de la `vie` de notre personnage, la `vitesse de déplacement` de celui-ci, voir même comment le téléporter n'importe où dans notre monde, comment faire ? nous allons voir ça ensemble.

## Les différents types de cheats.

Il existe 3 types de cheats (logiciels de triche) au total, nous allons les voir ensemble.

* Les externals
* Les internals
* Les semi-externals

## Les Externals

Les cheats de type `external` ont pour vocation de modifier la mémoire du jeu ciblé de façon externe, ici on parle d'un logiciel de triche qui va ouvrir le processus du jeu en question et modifier des pointers à la volée.

En gros quand vous utilisez ce type de cheats, vous allez utiliser des fonctions comme `ReadProcessMemory` et des `WriteProcessMemory` ou des équivalents pour lire des pointers et modifier leurs valeurs, via le user-land (API Win32) ou kernel-land (techniques plutôt avancées à ce moment là mais c'est possible), à voir, libre à vous d'imaginer toutes les possibilités possibles.

Exemple, dans mon jeu, il y a un pointer de type `integer` ou `float` qui va définir la vie de mon personnage, il suffit de réécrire cette valeur en question qui est souvent de `100` en integer ou `100.0f` pour un float dans la plupart des jeux afin de regénérer votre vie.

## Les Internals

Les internals sont différents des externals, car il s'agit là d'injecter une librairie de type `DLL` dans un jeu, afin d'obtenir le context courant de celui-ci et d'accéder aux fonctions internes du jeu, chose qui n'est pas possible via un cheat de type external de base.

Exemple avec GTA5, il fonctionne avec le moteur de jeu `Rage Engine`, et spoiler alert, ça ne sera plus Rage Engine pour GTA6 mais probablement Unreal Engine 5 comme certains s'en doutent probablement car le remaster de la trilogie fonctionne sous Unreal Engine 4.

Donc il est possible d'accéder aux fonctions internes du moteur de jeu via une librairie dite `injectée`.

## Les semi-externals

Les externals à la base comme vous l'aurez compris ne peuvent pas accéder aux fonctions internes du jeu ciblé car il s'agit seulement d'intéragir avec la mémoire du jeu de façon `externe`.

Ils sont souvent moins détectés que les internals mais peuvent parfois prendre plus de temps à développer contrairement aux internals (des jeux ont leur propres API internes comme la Rage Engine afin de faciliter le travail des développeurs donc c'est du pain bénit pour un internal cheat)

Le but ici est en fait d'injecter une librairie dans le jeu ciblé, et de faire communiquer la librairie injectée qui elle aura plus facilement accès aux fonctions internes du jeu afin de communiquer au programme externe l'information souhaitée, comme une API en gros mais qui fonctionne via des `Named Pipes` ou autres.

## Okay, c'est cool mais tu vas montrer quoi dans ton article ?

En fait pour créer un cheat sur un jeu vidéo, le mieux, c'est de connaître sous quel moteur graphique le jeu tourne dans un premier temps, est-ce que c'est du directx10, directx11, directx12 ou du Vulcan ?

Une fois qu'on connaît le moteur graphique (vous pouvez facilement le trouver dans les options du jeu), que voulez vous faire ?

Vous avez 3 choix à ce moment là, partir sur un internal, un external, ou un semi external.

Pour ma part, j'aime les 3 en fonction de mes besoins, mais pour GTA5 j'ai préféré l'internal au départ.

En gros j'injectais une librairie dll dans le jeu directement, pour accéder à l'API interne du Rage Engine.

En fait dans la mémoire quand vous allez étudier le fonctionnement de la rage engine, vous allez comprendre qu'il s'agit d'une API pour les développeurs de GTA accessible à un pointer précis dans la mémoire, auquel le client a dans tous les cas accès.

Que se passe t'il si je modifie mon véhicule en local avec la rage engine ? est-il visible seulement en mode solo ou est-il visible en multijoueur ?

La réponse est simple, il s'agit d'un jeu en peer2peer.

Ce que vous modifiez en tant que client simple via un WriteProcessMemory ou autre est directement répercuté sur les autres joueurs.

Et là vous allez me dire `Mais LOL ? donc si je fais spawn une voiture depuis mon jeu, les autres joueurs peuvent la voir ?` Bah ouais :)

## Allons plus loin

Si vous voulez développer un cheat `external`, vous aurez juste besoin de faire une interface graphique (GUI) en C# et accéder à la mémoire pour modifier les pointers souhaités (ReadProcessMemory / WriteProcessMemory).

Si vous voulez développer un cheat `internal`, vous aurez besoin de hook des fonctions de DirectX (la librarie [Kiero](https://github.com/Rebzzel/kiero) le fait très bien) afin d'afficher votre propre menu en C/C++ directement en jeu afin d'accéder à la mémoire ou directement aux fonctions internes du jeu.

Et si vous voulez implémenter votre cheat sous forme d'API afin d'exécuter les fonctions internes depuis n'importe quel logiciel sans rien injecter, à ce moment-là vous pouvez injecter une DLL qui pourra communiquer via des Named Pipes afin d'exécuter des fonctions internes du jeu depuis un programme externe, il s'agit à ce moment-là d'un `semi-external`.

## Un cas concret

Le fils de ma compagne joue à GTA, le problème c'est qu'il est mauvais en anglais car un peu jeune, donc plusieurs problèmes :

* Les cheats connus sont en anglais
* On sait pas ce qu'il y a dedans, un malware peut être facilement déployé dans le cheat
* On doit se contenter des fonctions implémentés, c'est cool, mais si on a envie de faire des trucs plus fun ?

Donc la solution pour moi, ça a été de recoder un cheat moi même, même si je suis en capacité d'en faire pour le mode en ligne, j'ai décidé de mettre des limitations et qu'il puisse pas l'utiliser en ligne.

Puis bon, je l'ai fais en bon français pour le coup, voyez par vous même.

![Sh0ckFR Game Hacking Regis](/images/cheat1.jpg)


Ici un exemple avec `voiture qui flotte` :

![Sh0ckFR Game Hacking Regis](/images/cheat2.jpg)

Bref comment il fonctionne :

* Je hook directx11 avec la librairie kiero pour afficher mon propre menu par dessus directx.
* Je récupère l'API de la rage engine nommée `natives table` chez les développeurs de Rockstar.
* J'exécute chaque fonction interne via cette table de fonctions internes afin de développer comme si j'étais un développeur de GTA5.

Enfin si vous êtes curieux à propos de la rage engine, il faut savoir que chaques fonctions internes sont nommées `natives` par les développeurs Rockstar, et qu'il est possible de trouver ses natives relativement facilement avec un peu de recherches, enfin il existe un morceau de code nommé `natives caller` qui permet de les exécuter.

## Si je n'ai pas accès à des fonctions internes, que faire ?

À ce moment-là, vous pouvez utiliser cheat engine sur les versions hors ligne des différents jeux vidéos (hors ligne = pas d'anticheat actif la plupart du temps, ce qui est le cas pour GTA).

Grâce à cheat engine, il est possible de récupérer les différents pointers comme la vie dans un premier temps.

Et vu que `la vie (the health)` appartient forcément à une entité `character` en programmation, cela représente grosse modo une classe `character` avec un attribut `health` mais dans ces attributs, il sera aussi possible d'accéder à d'autres attributs comme `characterposX` pour la position X, `characterposY` pour la position Y, voir même un attribut `colorBody` ou tout ce que vous pouvez imaginer.

Dans ce cas pourquoi ne pas récupérer la classe complète du personnage au lieu de seulement le pointer concernant la vie ?

C'est là que [Reclass.NET](https://github.com/ReClassNET/ReClass.NET) rentre en jeu.

Ce programme permet notamment d'afficher des structures de données depuis un pointer et ainsi retrouver la classe complète d'une entité comme un joueur, une voiture ou tout autre objet et d'en récupérer le code C++/C# pour accéder à ces structures directement dans votre code.

Voici 2 tutorials qui expliquent le fonctionnement de tout ça sur le jeu Assault Cube :

* https://www.youtube.com/watch?v=fvv8IJGke1Q
* https://www.youtube.com/watch?v=HZsnoUWK4Do

## Combien de temps pour apprendre tout ceci ?

Il faut compter à peu près 6 mois à un an environ avec quand même un bon bagage technique en programmation, il faut savoir que personne ne vous mâchera le travail et que la communauté du game hacking est plutôt élitiste voir toxique malheureusement, c'est pour cette raison que j'ai décidé d'écrire cet article pour aider un peu les débutants.

Des sites référents sont cependant très utilisés pour apprendre, notamment :

* https://guidedhacking.com/
* https://www.youtube.com/c/GuidedHacking
* https://www.unknowncheats.me/

Il faut aussi savoir que c'est typiquement le même mode de fonctionnement pour tous les jeux, et le développement de logiciels de triche est très proche du développement de malwares donc c'est une bonne base si vous voulez faire du red teaming et vous orienter sur le dev de payloads.

## Quid des autres jeux en ligne, sont-ils aussi peu fiables ?

La majorité sont aussi peu fiables oui, malgré les solutions anti-cheats existantes, des développeurs de logiciels de triche arrivent toujours avec plus ou moins de temps à contourner les protections en place.

Dans le cas de GTA5 par exemple, il suffit d'éviter d'appeler certaines fonctions `natives` pour contourner l'anti-cheat en place car il y a des checks sur ces natives côté serveur.

C'est aussi le cas sur des jeux présents sur Steam et autres plateformes, même si des progrès arrivent petit à petit à ce niveau là, sachez que de ma propre expérience, les jeux de tir en ligne sont extrêmement touchés par ce problème.

## Conclusion

Je vous laisse avec une vidéo du cheat développé spécialement pour le petit, qui je vous le rassure, n'est pas utilisable en ligne.

[https://www.youtube.com/watch?v=MmEvIvntXug](https://www.youtube.com/watch?v=MmEvIvntXug)

Si bien sûr vous souhaitez que j'aille plus loin dans l'écriture d'articles de ce type là, laissez le moi savoir.

EDIT: Suite à cet article, on m'a proposé de publier du code non mis à jour, je préfère que vous me contactiez dans un premier temps sur Twitter ou autre, et à ce moment-là je pourrais vous donner accès ou non suivant ce que vous faites comme activité.
