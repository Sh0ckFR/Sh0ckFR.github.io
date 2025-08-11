+++
author = "Sh0ckFR"
title = "Régis fait son propre C2"
date = "2025-08-11"
description = "Régis fait son propre C2"
tags = [
    "redteam", "c2",
]
+++

Hello tout le monde, ça fait un moment ^^

Aujourd'hui on va parler C2, comment bien penser son propre C2 avant de le développer car je me suis lancé sur le sujet, et après beaucoup beaucoup trop d'erreurs, j'ai maintenant une base solide sur le sujet car c'est probablement un des sujets les plus techniques à réaliser en programmation et je pèse mes mots, ne vous lancez pas là dedans si vous n'avez pas un très bon niveau en programmation sauf si vous aimez le challenge :')

## Petit Historique

![Sh0ckFR C2 dev](/images/c2-1.png)

Déjà il faut se rendre compte que la notion de C2 n'est pas nouvelle, effectivement comme vous pouvez le voir en screenshot, il s'agit de NetBus, le tout premier vrai `command and control` créer en 1997 par le Suédois Carl-Fredrik Neikter.

Ensuite Back Orifice a vu le jour, qui a été développé par le célèbre groupe de hackers Cult of the Dead Cow (cDc).

Puis ensuite Sub7 en 1999 qui était un C2 plus avancé que Back Orifice mais surtout utilisé par des scripts kiddies de l'époque.

Par la suite, les gens ont connu ProRat en 2002, à l'époque c'était surtout des logiciels rats pour faire chier son prochain clairement (coucou je t'ouvre le lecteur cd-rom à distance je suis trop un leet).

En 2008, DarkComet est ensuite sorti, par Jean-Pierre Lesueur (DarkCoderSC), tellement bien conçu pour l'époque que le régime syrien l'a utilisé contre des dissidents.

Le développeur pour des raisons éthiques, suite à l'usage politique de son C2 avait décidé d'arrêter son développement, ce que je trouve totalement admirable (merci DarkCoderSC pour ça, t'es le goat, la bise à toi toussa -> Oui on s'est fréquenté un peu IRL, cool guy).

![Sh0ckFR C2 dev](/images/c2-2.png)

Puis ensuite la référence absolue des C2 pour l'époque, est sorti en 2012, il s'agit de Cobalt-Strike développé par Raphael Mudge qui est encore vendu aujourd'hui car effectivement il s'agit d'un C2 professionnel orienté pour les équipes Red Team.

![Sh0ckFR C2 dev](/images/c2-3.png)

Aujourd'hui, la plupart des équipes Red Team possèdent différents C2, que ça soit Cobalt-Strike, BruteRatel, Mythic C2, Havoc ou bien d'autres.

Chacuns ont leurs avantages et inconvéniants mais ce qui est intéressant pour une équipe Red Team, c'est de varier ses TTP (Tactics, Techniques and Procedures).

Ce qui veut dire entraîner les équipes défensives à voir des choses nouvelles à chaque fois pour être capable de mettre en place des règles de sécurité polyvalentes et efficaces par rapport aux nouvelles menaces et aux nouveaux modes d'attaques.

Il s'avère que plus on utilise un C2 et plus il est utilisé dans d'autres équipes red team autour du globe, plus il est facilement identifiable car les équipes défensives arrivent à identifier leurs comportements ou les identifier via des particularités précises vu que c'est toujours le même logiciel vendu à des red teams.

Il y a parfois aussi des leaks qui sont distribués sur internet, crackés via des licences officielles et utilisées par des threat actors donc on peut retrouver facilement un Cobalt-Strike, un BruteRatel ou autre dans la nature en torrent dans une ancienne version qui peut permettre de détecter les nouvelles versions, bref c'est le jeu du chat et de la souris.

Il se trouve que dans le cadre d'une red team qui a déjà utilisée et qui connaît bien ces frameworks C2, elle peut décider de développer son propre C2 pour des accès initiaux voir une solution complète afin d'avoir une base complètement clean pour les antivirus et EDRs et customisable.

Dans notre équipe, c'est un peu notre cas, on apprécie utiliser des nouvelles méthodes donc dans ce cadre là, totalement encadré et légitime bien entendu, nous décidons de développer ce genre de solutions.

Nous ne pouvons pas bien entendu partager des sources complètes, car forcément utilisées par des threats actors ensuite.

Dans cet article, je vais cependant expliquer comment développer son propre C2 fonctionnel sans pour autant divulguer les sources complètes, ce qui sera certainement utile pour des équipes red team voulant faire de la R&D sur le sujet.

## Comment se compose l'architecture d'un C2

Concrêtement, de base un C2 est composé de quoi ?

* L'implant : Donc le binaire malveillant qui va communiquer toutes les X secondes vers différents domaines (CDNs ou autre), il se chargera via des requêtes POST par exemple de demander `coucou je dois exécuter quoi comme commande ?`

* Le serveur : Lui sera hébergé sur en local non exposé mais qui écoutera les demandes de l'implant via un redirector exposé sur le net (je reviendrais sur le sujet pour expliquer le principe d'un redirector) `coucou je vois ce que tu me demandes, actuellement le client t'a demandé d'exécuter telle action donc fais le et renvoie moi le résultat ensuite`

* Le client : Le client est l'interface graphique, non accessible depuis internet, il devra être connecté via un VPN par exemple au serveur, il aura pour but de dire au serveur `coucou je veux que l'implant de madame michue fasse telle action sur son poste et une fois que tu as la réponse, il faut que tu me la renvoie pour que je l'affiche pour le red teamer`

![Sh0ckFR C2 dev](/images/c2-4.png)

## De quoi est composé l'implant en lui même ?

De base il s'agit d'une grande `boucle while(1)` qui toutes les X secondes va demander via une requête POST la plupart du temps sur un redirector en lui demandant `coucou on t'a demandé à ce que je passe des commandes ?` si c'est le cas, il exécutera ces commandes et renverra ensuite le résultat par une autre requête POST `coucou j'ai exécuté ce que tu m'as dis, voici la réponse`.

En pseudo code rapide ça donnerait quelque chose du genre :

```cpp
while(1) {
    std::string command = POST(url, userAgent, postData);
    std::string result = exec(command);
    POST(url, userAgent, resultPostData);
    Sleep(TIMER + JITTER);
}
```

Alors là vous me direz, oui mais faut que ça soit chiffré un minimum et noyé dans la masse ?

Et bien vous avez raison, en fait la plupart du temps, ce que font les C2 classiques, c'est qu'ils mettent un POST prepend et un POST append avec du chiffrement de leur payload et de l'encodage pour éviter les problèmes d'UTF-8 par exemple, ils encodent ça en base64 ou ils font du HEX, il y a plein de méthodes différentes pour gérer ce cas là mais concrêtement la requête POST dans le body pourrait donner quelque chose comme :

```
id=1&fake_cache_data=BLABLA&cache_data=PAYLOAD&output=json
```

Où `fake_cache_data` serait utilisé pour tromper des équipes SOC avec une fausse payload, et la vraie payload se trouverait par exemple dans `cache_data`.

Il y a encore plus surnois, c'est de découper la payload en plusieurs POST avec des leurres à côté, mais je n'ai jamais encore vu trop ça dans un C2 public ou commercial, ça reste compliqué à mettre en place niveau code en réalité.

Il est aussi possible et ce qu'on voit la plupart du temps, d'avoir l'implant qui communiquer son `SERVER_RESPONSE_PREPEND` et `SERVER_RESPONSE_APPEND` c'est concrêtement la même chose pour que le `POST prepend` et `POST append` mais ça concerne la réponse du serveur, exemple :

```html
<html><body><p>PAYLOAD</p></body></html>
```

La PAYLOAD se trouve à ce moment là entre son `SERVER_RESPONSE_PREPEND` et `SERVER_RESPONSE_APPEND` qui sont les balises html de réponse du serveur.

Ici le `SERVER_RESPONSE_PREPEND` serait :

```html
<html><body><p>
```

Et le `SERVER_RESPONSE_APPEND` serait :

```html
</p></body></html>
```

## De quoi est composé le serveur ?

Le serveur attend et traite les requêtes de l'implant, il a une base de données disponible qui lui permet d'enregistrer les commandes qu'il a reçu, les commandes qu'il a pas encore traîté "is_executed?" et les résultats si "is_executed est égal à False, l'implant exécute la commande et renvoie le résultat, à ce moment là is_executed passe à true avec le résultat de la commande" donc l'implant sait du coup qu'il ne doit plus la traiter.

## De quoi est composé le client ?

Le client est l'interface graphique (ou CLI peu importe) d'un red teamer, qui ne sera jamais exposée sur internet, elle sert à communiquer directement les commandes ou informations que l'implant devra exécuter via le serveur.

Ce client va interroger le serveur pour avoir la liste des commandes passées sur celui-ci et voir si il a obtenu une réponse pour chacune d'entre elles et les afficher.

## C'est quoi un redirector ?

Un redirector est en gros un serveur web type apache ou nginx la plupart du temps qui va attendre des requêtes en http ou https, et qui va les rediriger en interne via un VPN par exemple à votre serveur C2, vu qu'on ne veut pas exposer le serveur C2 sur le net, le redirector lui est exposé mais on s'en moque pas mal au final car ça reste un serveur web classique, le plus souvent hébergé via un domaine catégorisé, et ce domaine là pour éviter d'apparaître trop souvent en fonction des hits des pics de connexions, peut être camouflé via des CDNs.

## Qu'est-ce que le Timer et le Jitter ?

Concrêtement le `Timer` est le nombre de secondes que vous voulez que l'implant fasse une requête au redirector, qui lui va transmettre la demande au serveur C2 hébergé en interne.

Le `Jitter` est souvent noté en pourcentage, en gros pour éviter d'avoir un `Timer` fixe et récurrent, le `Jitter` lui va rajouter un délai supplémentaire genre 20 ou 50% du temps en plus consacré au `Timer` de façon random, de ce fait, vos requêtes de l'implant deviennent plus aléatoires, ce qui empêche des détections par rapport à la récurrence des requêtes (si votre implant communique toutes les 5 secondes de façon fixe c'est suspect par exemple mais si votre implant communique toutes les 5 secondes + un pourcentage aléatoire, ça l'est moins).

## À cette étape là qu'est que cela nous explique ?

À cette étape là, vous pouvez comprendre à minima que la base de données est entièrement côté serveur pour 90% des données.

Côté client, vous allez juste retrouver les différents serveurs auquel vous vous êtes connecté mais rien de plus.

`Pourquoi ?`

Car si vous voulez des interactions multi utilisateurs, il faut bien comprendre que c'est votre serveur, et non vous en tant que client qui allait synchroniser les données, les clients se connectent au serveur, mais c'est à lui de prendre en compte chaque requête de chaque client.

De ce fait, vous n'aurez pas de problèmes de synchronisation entre clients si vous avez bien pensé à ça.

## Une partie OPSEC des commandes de base

Oui c'est bien beau d'implémenter les commandes de bases `cd` `ls` ou whatever, il y a des choses à comprendre, notamment le fait de ne pas utiliser des commandes trop connues dans votre C2 style `whoami` pour ce genre de commandes, il est préférable de produire l'équivalent sans appeler les commandes systèmes de base qui peuvent être très monitorées.

Pour cela, il est plus intéressant de se passer sur du code open source Windows comme ReactOS pour éviter de déclencher des alertes de base, mieux comprendre la source ReactOS vous permettra de comprendre qu'il est possible de faire des actions vraiment offensives sans être détecté, pour cela il y a pas de secret, il faut se plonger dans la source et comprendre les mécanismes.

C'est le cas par exemple pour `whoami`, ou bien faire un dump lsass sans appeler la fonction `minidumpwritedump` directement pour dumper lsass, des multiples exemples sont présents.

Des projets OpenSource plus évolués sont disponibles sur github, plus vous comprenez les potentielles détections côté SOC, plus vous serez efficace en red team et donc en capacité de développer un outil C2 totalement opérationnel.

Cela demande clairement d'avoir au moins touché à un SIEM une fois, donc avoir une petite expérience blue à minima et voir la télémétrie présente dans l'EDR (dans le bouzin comme on dit).

## La fonctionnalité la plus hardcore dans un C2, le socks5.

On y vient, la fonctionnalité socks5 est très utilisée nos jours par des red teamers, car elle permet de créer un tunnel socks5 pour utiliser le réseau disponible d'une machine distante infectée afin de joindre d'autres machines qui ne sont pas accessibles depuis Internet.

`Comment ça fonctionne ?`

Imaginez vous avez votre implant qui fonctionne sur une machine nommée `toto`, vous avez accès à cette machine mais pas au serveur `tata` directement qui est accessible que depuis le réseau interne de `toto`.

Le but du socks5 est d'utiliser la machine `toto` dont vous avez accès en passant par le réseau interne pour accéder au réseau interne et donc accéder au serveur `tata`.

Au final vous utiliser la connexion de `toto` en socks5 pour atteindre le serveur interne `tata`.

Le problème qu'il se pose, c'est que votre implant communique avec vous seulement via des requêtes POST de base.

Nous il nous faut un canal socks5 au complet, fonctionnel et sans crashes.

Je mettrais à jour cet article pour expliquer tout ça dans une seconde partie car on rentre clairement dans l'aspect technique avancé de la chose.

Des bisous :)