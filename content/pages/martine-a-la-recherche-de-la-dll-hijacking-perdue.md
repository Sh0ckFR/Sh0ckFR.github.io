+++
author = "Sh0ckFR"
title = "Martine à la recherche de la DLL Hijacking perdue"
date = "2022-03-15"
description = "Martine à la recherche de la DLL Hijacking perdue"
tags = [
    "red-team",
]
+++

Tout d'abord, vous me pardonnerez pour le titre de l'article, effectivement, durant mes recherches sur du `DLL Hijacking` un peu plus poussé que dans cet article, j'avais l'impression que Windows me faisait tourner en bourique, d'où ce titre peu glorieux car j'ai fail de nombreuses fois.

Nous allons tout de même voir la base dans cet article, le but ici est de vous donner un aperçu de ce qu'est le DLL Hijacking et en quoi il peut être utile durant un exercice `Red-Team`, bien entendu tout comme moi, vous pourrez ensuite aller plus loin en vous intéressant au sujet, la technique n'étant pas nouvelle, il y a tout de même pas assez de ressources à mon sens sur le sujet et donc beaucoup à faire.

![Sh0ckFR DLL Hijacking Martine](/images/martine-dll-hijacking.jpg)

## Qu'est-ce que le DLL Hijacking ?

Sous Microsoft Windows, les applications ont besoin de tourner via des dépendances qui ont généralement l'extension `.dll` pour `Dynamic Link Libraries`.

Pour vous faire une idée, une DLL se présente sous cette forme en C, ici une simple DLL qui va créer un Thread et afficher une MessageBox :

```c
#include <windows.h>

int Main() {
    MessageBoxW(0, L"hello c'est moi Martine", L"Hello", 0);
    return 1;
}

BOOL APIENTRY DllMain(HMODULE hModule,
    DWORD  ul_reason_for_call,
    LPVOID lpReserved
)
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        CreateThread(NULL, NULL, (LPTHREAD_START_ROUTINE)Main, NULL, NULL, NULL);
        break;
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```

C'est bien beau mais j'en fais quoi de ma DLL Martine ?

Pour l'exemple, imaginez un programme `toto.exe` qui va au lancement charger sa DLL `tata.dll` pour effectuer une opération comme créer un fichier sur le disque avec une fonction du genre `CreateFile`.

Le but du DLL Hijacking est de se faire passer pour `tata.dll` via notre propre fichier DLL et ainsi exécuter notre code à la place de la fonction `CreateFile`.

Ce qui permet à un attaquant de d'utiliser un programme légitime voir même certifié par une autorité afin d'exécuter du code malveillant, cela ne vous rappelle rien ?

Un petit indice : https://cyberint.com/blog/research/solarwinds-supply-chain-attack/

## Cas concret

Nous allons prendre comme exemple `nslookup.exe` qui est un binaire natif présent sur Windows 10 et qui a été modifié pour la dernière fois en 2019.

Pour tester un DLL Hijacking, nous allons copier l'exécutable dans un répertoire vide et tenter de l'exécuter une première fois :

![Sh0ckFR DLL Hijacking 1](/images/dll-hijacking1.png)

Dans mon cas je peux voir le nom de fournisseur d'accès Internet et l'adresse de ma box.

Et voilà ! fin de l'article merci d'être passé ! non je déconne :)

Nous allons maintenant nous intéresser aux fonctions dont à besoin le binaire pour fonctionner, pour cela nous allons utiliser l'utilitaire `dumpbin.exe` disponible avec Visual Studio via le `x64 Native Command Tools`.

```
dumpbin /imports nslookup.exe
```

Nous avons comme résultat :

```
Microsoft (R) COFF/PE Dumper Version 14.29.30136.0
Copyright (C) Microsoft Corporation.  All rights reserved.


Dump of file nslookup.exe

File Type: EXECUTABLE IMAGE

  Section contains the following imports:

    msvcrt.dll
             14000D448 Import Address Table
             1400113D8 Import Name Table
                     0 time date stamp
                     0 Index of first forwarder reference

                         4A0 putchar
                         4B9 sprintf_s
                         461 gmtime
                         432 exit
                         363 _vsnprintf
                         304 _strnicmp
                         439 fflush
                         486 malloc
                         ...
    DNSAPI.dll
             14000D238 Import Address Table
             1400111C8 Import Name Table
                     0 time date stamp
                     0 Index of first forwarder reference

                          45 DnsFreeConfigStructure
                          75 DnsQueryConfigAllocEx
    ...
```

Comme vous pouvez le voir, beaucoup de dépendances demandées par le binaire, notamment des librairies classiques qu'on retrouve un peu partout comme `msvcrt.dll` dont les fonctions ne peuvent pas vraiment être détournées.

Cependant on peut voir que des librairies comme `DNSAPI.dll` sont disponibles et que 2 fonctions `DnsFreeConfigStructure` et `DnsQueryConfigAllocEx` sont demandées par le programme.

Windows contient une clé de registre nommée `SafeDllSearchMode` disponible dans `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager`.

Quand notre programme est lancé, Windows va dire au programme en fonction de si cette clé est active ou non d'aller chercher ses dépendances dans cet ordre.

Si le `SafeDllSearchMode` est actif :

- 1. Répertoire à partir duquel l’application a été chargée.
- 2. Répertoire du système. Utilisez la fonction GetSystemDirectory pour récupérer le chemin d’accès à ce répertoire.
- 3. Répertoire système 16 bits. Aucune fonction n’obtient le chemin d’accès de ce répertoire, mais elle est recherchée.
- 4. répertoire Windows. Utilisez la fonction GetWindowsDirectory pour récupérer le chemin d’accès à ce répertoire.
- 5. Le répertoire actif.
- 6. Répertoires répertoriés dans la variable d’environnement PATH. Notez que cela n’inclut pas le chemin d’accès par application spécifié par la clé de Registre App Paths . La clé chemins d’accès à l’application n’est pas utilisée lors du calcul du chemin de recherche de dll.

Si `SafeDllSearchMode` est désactivé :

- 1. Répertoire à partir duquel l’application a été chargée.
- 2. Le répertoire actif.
- 3. Répertoire du système. Utilisez la fonction GetSystemDirectory pour récupérer le chemin d’accès à ce répertoire.
- 4. Répertoire système 16 bits. Aucune fonction n’obtient le chemin d’accès de ce répertoire, mais elle est recherchée.
- 5. répertoire Windows. Utilisez la fonction GetWindowsDirectory pour récupérer le chemin d’accès à ce répertoire.
- 6. Répertoires répertoriés dans la variable d’environnement PATH. Notez que cela n’inclut pas le chemin d’accès par application spécifié par la clé de Registre App Paths . La clé chemins d’accès à l’application n’est pas utilisée lors du calcul du chemin de recherche de dll.

Vous pouvez retrouver cette information ici : https://docs.microsoft.com/fr-fr/windows/win32/dlls/dynamic-link-library-search-order

Nous remarquons donc que dans les 2 cas, le programme va chercher ses dépendances dans le répertoire où l'application a été chargée, il suffit donc de nommer notre dll `DNSAPI.dll` dans le même répertoire que `nslookup.exe` pour que celui-ci décide de la charger en priorité.

Testons donc de compiler notre DLL comme telle et lançons le programme.

![Sh0ckFR dll Hijacking 2](/images/dll-hijacking2.png)

Et là c'est le drame, ça ne fonctionne pas, avons-nous dit notre dernier mot Jean-Pierre ? pas forcément.

Testons de recompiler notre DLL en exportant la fonction `DnsFreeConfigStructure` :

```c
#include <windows.h>

int Main() {
    MessageBoxW(0, L"hello c'est moi Martine", L"Hello", 0);
    return 1;
}

BOOL APIENTRY DllMain(HMODULE hModule,
    DWORD  ul_reason_for_call,
    LPVOID lpReserved
)
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        CreateThread(NULL, NULL, (LPTHREAD_START_ROUTINE)Main, NULL, NULL, NULL);
        break;
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}

__declspec(dllexport) void DnsFreeConfigStructure() { Main(); }
```

On peut voir ici qu'on a rajouté la fonction `DnsFreeConfigStructure` via le code `__declspec(dllexport) void DnsFreeConfigStructure() { Main(); }` qui aura pour but de lancer notre fonction `Main()` si la fonction est bien utilisée par le programme au lancement.

![Sh0ckFR dll Hijacking 3](/images/dll-hijacking3.png)

Toujours un fail mais on a du mieux, on peut voir via le message d'erreur que le programme demande aussi la fonction `DnsQueryConfigAllocEx` pour se lancer, rajoutons la :

```c
#include <windows.h>

int Main() {
    MessageBoxW(0, L"hello c'est moi Martine", L"Hello", 0);
    return 1;
}

BOOL APIENTRY DllMain(HMODULE hModule,
    DWORD  ul_reason_for_call,
    LPVOID lpReserved
)
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        CreateThread(NULL, NULL, (LPTHREAD_START_ROUTINE)Main, NULL, NULL, NULL);
        break;
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}

__declspec(dllexport) void DnsFreeConfigStructure() { Main(); }
__declspec(dllexport) void DnsQueryConfigAllocEx() { Main(); }
```

Bingo Martine a trouvée la DLL Hijacking perdue dans `nslookup.exe` !

![Sh0ckFR dll Hijacking 4](/images/dll-hijacking4.png)

On peut voir que 2 MessageBox s'affichent donc que 2 fonctions sont exécutées, potentiellement `DnsFreeConfigStructure` et `DnsQueryConfigAllocEx` ou bien `DnsQueryConfigAllocEx` et `DllMain` qui est le point d'entrée de notre DLL.

Je vous donne la réponse, il s'agit de `DnsQueryConfigAllocEx` et `DllMain`.

`nslookup.exe` n'ayant pas de certificat Microsoft mais étant tout de même un binaire de Windows natif, cela peut être utile, cependant vu qu'une command line s'ouvre au lancement, il n'est malheureusement pas vraiment utilisable dans le cadre d'un exercice Red-Team.

Cependant beaucoup de binaires ne lancent pas forcément de fenêtres à l'exécution et sont parfois signés par les développeurs.

On peut imaginer exécuter un beacon `Cobalt-Strike` à la place de notre simple MessageBox dans une situation réelle.

## Mot de la fin

Comme vous avez pu le voir, le DLL Hijacking permet notamment d'exécuter du code malveillant via des programmes légitimes et permet donc d'augmenter la furtivité d'un attaquant en particulier si celui-ci passe par un binaire signé par un éditeur.

Cette méthode est utilisée par des opérateurs Red-Team mais également par des acteurs malveillants, il est également possible d'utiliser Process Monitor avec le filtre `Result` sur `NAME NOT FOUND` pour observer des binaires recherchant leurs DLL perdues.

https://pentestlab.blog/2017/03/27/dll-hijacking/

Notez également que certains programmes demandent des privilèges Administrateur pour être exécutés donc ne sont pas utilisables si vous possédez un simple compte utilisateur durant votre exercice.