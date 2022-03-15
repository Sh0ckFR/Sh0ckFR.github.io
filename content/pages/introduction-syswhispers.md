+++
author = "Sh0ckFR"
title = "Les différences entre Red Team et Pentest"
date = "2022-01-02"
description = "Introduction à Syswhispers1 et Syswhispers2"
tags = [
    "red-team",
]
+++

Introduction à Syswhispers1 et Syswhispers2

Bon, commençons par le commencement, Syswhispers c'est quoi ?

Il s'agit en fait d'une méthode permettant d'effectuer des appels dits "direct system calls".

L'instruction `syscall` est utilisée pour faire la transition entre le `User-Mode (UM)` et le `Kernel-Mode (KM)` en utilisant un identifiant 16-bits WORD depuis le registre EAX.

Cet identifiant étant un index dans la structure interne de la table `System Service Descriptor Table (SSDT)` dans la pile d'exécution du Kernel.

En temps normal, quand vous utilisez des fonctions user-land comme `WriteProcessMemory` qui font parties de l'`API Win32`, celles-ci passent par `ntdll.dll` et la fonction équivalente côté kernel-land `NtWriteVirtualMemory` est utilisée dans l'opération.

La plupart des antivirus / EDR se hookent principalement sur le module ntdll.dll pour analyser quelles fonctions vous utilisez.

Un hook correspond à un trampoline, cette méthode permet de se greffer entre le début d'exécution d'une fonction et sa fin afin d'observer ce qui se déroule durant celle-ci, un peu comme un `Man In The Middle (MITM)` mais pour une fonction dans un programme.

Voici les différentes possibilités de hook une fonction : https://www.unknowncheats.me/forum/general-programming-and-reversing/154643-different-ways-hooking.html

Et donc si vous utilisez une fonction qui utilise `NtWriteVirtualMemory` et que celle-ci est hookée, votre EDR sera en capacité d'avoir plus de visibilité pour réaliser des analyses dynamiques durant l'exécution et potentiellement émettre une alerte pour l'équipe Blue-Team si celui-ci observe un comportement suspect.

Pour pallier à ce problème, une équipe de chercheurs s'est basée sur la table des syscalls (system calls) récupérée par `j00ru` du Google Project Zero à l'origine dans la table `System Service Descriptor Table (SSDT)`.

Entre les versions de Windows, les numéros des syscalls varient, la table est visible sur cette page :

https://j00ru.vexillium.org/syscalls/nt/64/

Syswhispers qui est disponible sur cette page https://github.com/jthuraisamy/SysWhispers permet de générer un stub qui va réimplémenter les syscalls en fonction de cette table et de la version Windows souhaitée.

La différence notable entre le PoC de Dumpert https://github.com/outflanknl/Dumpert et Syswhispers, est que Syswhispers ne va pas utiliser la fonction `RtlGetVersion` pour obtenir la version de Windows mais va directement interroger le `Process Environment Block (PEB)` qui est une méthode un peu plus discrète que la précédente.

Du coup comment utiliser Syswhispers ? le project est très accessible, la première étape est de générer le stub assembleur qui réimplémente les différents syscalls :

https://github.com/jthuraisamy/SysWhispers#command-lines

Ce qui va vous donner un fichier .asm et un .h à importer dans votre projet, si vous êtes sur Visual Studio, n'oubliez pas d'activer MASM (right click sur votre projet -> build customisations -> on coche la petite case MASM).

Ensuite comme le montre le projet GitHub, voici l'exemple d'un `CreateRemoteThread` pendant une injection de DLL sans utiliser Syswhispers :

```cpp
#include <Windows.h>

void InjectDll(const HANDLE hProcess, const char* dllPath)
{
    LPVOID lpBaseAddress = VirtualAllocEx(hProcess, NULL, strlen(dllPath), MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
    LPVOID lpStartAddress = GetProcAddress(GetModuleHandle(L"kernel32.dll"), "LoadLibraryA");
   
    WriteProcessMemory(hProcess, lpBaseAddress, dllPath, strlen(dllPath), nullptr);
    CreateRemoteThread(hProcess, nullptr, 0, (LPTHREAD_START_ROUTINE)lpStartAddress, lpBaseAddress, 0, nullptr);
}
```

Avec Syswhispers (donc les syscalls), cela donnera :

```cpp
#include <Windows.h>
#include "syscalls.h" // Import the generated header.

void InjectDll(const HANDLE hProcess, const char* dllPath)
{
    HANDLE hThread = NULL;
    LPVOID lpAllocationStart = nullptr;
    SIZE_T szAllocationSize = strlen(dllPath);
    LPVOID lpStartAddress = GetProcAddress(GetModuleHandle(L"kernel32.dll"), "LoadLibraryA");
   
    NtAllocateVirtualMemory(hProcess, &lpAllocationStart, 0, (PULONG)&szAllocationSize, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
    NtWriteVirtualMemory(hProcess, lpAllocationStart, (PVOID)dllPath, strlen(dllPath), nullptr);
    NtCreateThreadEx(&hThread, GENERIC_EXECUTE, NULL, hProcess, lpStartAddress, lpAllocationStart, FALSE, 0, 0, 0, nullptr);
}
```

Ce qui a changé, c'est que `VirtualAllocEx` a été remplacé par le syscall `NtAllocateVirtualMemory`, `WriteProcessMemory` par `NtWriteVirtualMemory` et `CreateRemoteThread` par `NtCreateThreadEx`.

Et là vous me direz "mais attend tonton Sh0ck, certains arguments sont différents ?"

Ha bah oui, pour utiliser les APIs non documentées, c'est à vous de chercher les bons arguments à utiliser en fonction de la non documentation (oui elle est pas censée être documentée) mais elle est quand même disponible ici :

http://undocumented.ntinternals.net/

Prenons l'exemple de l'appel de `VirtualAllocEx` documenté sur le site de Microsoft :

https://docs.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualallocex#syntax

On peut voir qu'elle prend en arguments _In_ un `HANDLE hProcess`, en _In_opt_ une `LPVOID lpAddress`, en _In_ un `SIZE_T dwSize`, et 2 DWORD `flAllocationType et flProtect`.

En cherchant dans la documentation non documentée on tombe sur la description de l'appel `NtAllocateVirtualMemory` :

```cpp
NtAllocateVirtualMemory(
  _In_ HANDLE               ProcessHandle,
  _In_out_ PVOID            *BaseAddress,
  _In_ ULONG                ZeroBits,
  _In_out_ PULONG           RegionSize,
  _In_ ULONG                AllocationType,
  _In_ ULONG                Protect );
```

Ce qui nous permet d'adapter notre code avec les bons arguments attendus, et c'est valable pour toutes les autres fonctions.

Si jamais vous débutez et que vous ne comprenez pas comment adapter ces fonctions, je recommande de lire des tutoriaux sur du game hacking, c'est de cette façon que j'ai appris, c'est ludique et ça permet de mieux comprendre comment fonctionne l'allocation de la mémoire sur Windows et ça reste relativement proche du développement de malwares.

Maintenant que nous avons vu Syswhispers première version, quelle est la différence avec la version 2 disponible ici :

https://github.com/jthuraisamy/SysWhispers2

Ou la version pour x86 disponible ici : https://github.com/mai1zhi2/SysWhispers2_x86

Cela reste assez similaire sauf que cette version n'utilise plus la table de j00ru mais elle utilise à la place la méthode "Sorting by System call Address" de `modexpblog` :

https://www.mdsec.co.uk/2020/12/bypassing-user-mode-hooks-and-direct-invocation-of-system-calls-for-red-teams/

Cette méthode permet de parser l'`Export Address Table (EAT)` du module ntdll.dll afin de localiser toutes les fonctions qui commencent par "Zw" et remplace "Zw" par "Nt" avant de générer les hashs des différentes fonctions afin de les appeler ensuite.

Ce qui permet de ne pas avoir à préciser la version de Windows pendant la génération du stub.

Syswhispers2 a aussi le grand avantage de ne pas charger une nouvelle copie de ntdll.dll.

Il existe aussi 2 versions pour utiliser Syswhispers dans des BOF Cobalt-Strike :

Une développée par `Outflank` pour Syswhispers1 : https://github.com/outflanknl/InlineWhispers

Et la seconde version pour Syswhispers2 développée par moi même : https://github.com/Sh0ckFR/InlineWhispers2

D'autres variantes sont également disponibles publiquement sur GitHub.

## Crédits

Un grand merci à Paul Laîné (@am0nsec) pour la relecture de cet article https://twitter.com/am0nsec