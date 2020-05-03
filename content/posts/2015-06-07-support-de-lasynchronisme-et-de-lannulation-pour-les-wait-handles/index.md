---
layout: post
title: Support de l’asynchronisme et de l’annulation pour les wait handles
date: 2015-06-07T00:00:00.0000000
url: /2015/06/07/support-de-lasynchronisme-et-de-lannulation-pour-les-wait-handles/
tags:
  - async
  - C#
  - waithandle
categories:
  - Code sample
---


Le .NET Framework fournit un certain nombre de primitives de synchronisation bas niveau. Les plus couramment utilisées sont appelées “wait handles”, et héritent de la classe `WaitHandle` : `Semaphore`, `Mutex`, `AutoResetEvent` et `ManualResetEvent`. Ces classes existent depuis .NET 2.0 (voire 1.1 pour certaines), mais elles n’ont pas beaucoup évolué depuis, ce qui fait qu’elles ne supportent pas des fonctionnalités introduites plus tard et devenues très courantes. En particulier, elles ne supportent pas l’attente asynchrone, ni l’annulation de l’attente. Heureusement, il est assez facile d’ajouter ces fonctionnalités via des méthodes d’extension.

### Annulation

Commençons par le plus facile : l’annulation de l’attente. Dans certains cas, on voudrait pouvoir passer un `CancellationToken` à `WaitHandle.WaitOne`, mais aucune des surcharges ne le supporte. Notez que des variantes plus récentes de certaines primitives de synchronisation, comme `SemaphoreSlim` et `ManualResetEventSlim`, supportent l’annulation ; cependant elles ne sont pas appropriées dans toutes les situations, car elles sont conçues pour les cas où les temps d’attente sont très courts.

Un `CancellationToken` expose un `WaitHandle`, qui est signalé quand l’annulation est demandée. On peut tirer parti de cela pour implémenter l’attente asynchrone sur un wait handle :

```
public static bool WaitOne(this WaitHandle handle, int millisecondsTimeout, CancellationToken cancellationToken)
{
    int n = WaitHandle.WaitAny(new[] { handle, cancellationToken.WaitHandle }, millisecondsTimeout);
    switch (n)
    {
        case WaitHandle.WaitTimeout:
            return false;
        case 0:
            return true;
        default:
            cancellationToken.ThrowIfCancellationRequested();
            return false; // never reached
    }
}
```

On utilise `WaitHandle.WaitAny` pour attendre que le wait handle d’origine ou celui du jeton d’annulation soit signalé. `WaitAny` renvoie l’index du premier wait handle qui a été signalé, ou  `WaitHandle.WaitTimeout` si un timeout s’est produit avant que l’un des wait handles ne soit signalé. On a donc 3 résultats possibles :

- un timeout s’est produit : on renvoie false (comme la méthode `WaitOne` standard) ;
- le wait handle d’origine a été signalé en premier : on renvoie true (comme la méthode `WaitOne` standard) ;
- le wait handle du jeton d’annulation a été signalé en premier : on lance une `OperationCancelledException`.


Pour compléter, on peut rajouter quelques surcharges pour les cas d’utilisation courants :

```
public static bool WaitOne(this WaitHandle handle, TimeSpan timeout, CancellationToken cancellationToken)
{
    return handle.WaitOne((int)timeout.TotalMilliseconds, cancellationToken);
}
 
public static bool WaitOne(this WaitHandle handle, CancellationToken cancellationToken)
{
    return handle.WaitOne(Timeout.Infinite, cancellationToken);
}
```

Et voilà, on a maintenant une méthode `WaitOne` qui supporte l’annulation !

### Attente asynchrone

Maintenant, comment faire pour attendre un wait handle de façon asynchrone ? C’est un peu plus difficile. Ce qu’on veut ici est une méthode `WaitOneAsync` qui va renvoyer un `Task<bool>` (et pendant qu’on y est, autant inclure aussi le support de l’annulation). L’approche habituelle pour créer un wrapper `Task` pour une opération asynchrone non basée sur une tâche est d’utiliser un `TaskCompletionSource<T>`, c’est donc ce qu’on va faire. Quand le wait handle sera signalé, on mettra à true le résultat de la tâche; si un timeout se produit, on le mettra à false; et si le jeton d’annulation est signalé, on marquera la tâche comme annulée.

J’ai eu un peu de mal à trouver un moyen d’exécuter un delegate quand un wait handle est signalé, mais j’ai fini par tomber sur la méthode `ThreadPool.RegisterWaitForSingleObject`, qui sert précisément à ça. Je ne sais pas trop pourquoi elle est dans la classe `ThreadPool` ; il me semble que ça aurait eu plus de sens de la mettre dans la classe `WaitHandle`, mais je suppose qu’il y a une bonne raison.

Voilà donc ce qu’on va faire :

- créer un `TaskCompletionSource<bool>` ;
- enregistrer un delegate qui mettra le résultat de la tâche à true quand le wait handle sera signalé, ou à false si un timeout se produit avant, à l’aide de `ThreadPool.RegisterWaitForSingleObject` ;
- enregistrer un delegate pour marquer la tâche comme annulée quand le jeton d’annulation sera signalé, en utilisant `CancellationToken.Register` ;
- désenregistrer les deux delegates une fois que la tâche est terminée.


Voici l’implémentation :

```
public static async Task<bool> WaitOneAsync(this WaitHandle handle, int millisecondsTimeout, CancellationToken cancellationToken)
{
    RegisteredWaitHandle registeredHandle = null;
    CancellationTokenRegistration tokenRegistration = default(CancellationTokenRegistration);
    try
    {
        var tcs = new TaskCompletionSource<bool>();
        registeredHandle = ThreadPool.RegisterWaitForSingleObject(
            handle,
            (state, timedOut) => ((TaskCompletionSource<bool>)state).TrySetResult(!timedOut),
            tcs,
            millisecondsTimeout,
            true);
        tokenRegistration = cancellationToken.Register(
            state => ((TaskCompletionSource<bool>)state).TrySetCanceled(),
            tcs);
        return await tcs.Task;
    }
    finally
    {
        if (registeredHandle != null)
            registeredHandle.Unregister(null);
        tokenRegistration.Dispose();
    }
}
 
public static Task<bool> WaitOneAsync(this WaitHandle handle, TimeSpan timeout, CancellationToken cancellationToken)
{
    return handle.WaitOneAsync((int)timeout.TotalMilliseconds, cancellationToken);
}
 
public static Task<bool> WaitOneAsync(this WaitHandle handle, CancellationToken cancellationToken)
{
    return handle.WaitOneAsync(Timeout.Infinite, cancellationToken);
}
```

Notez que les expressions lambda auraient pu utiliser la variable `tcs` directement ; cela rendrait le code un peu plus lisible, mais causerait la création d’une closure, donc pour éviter cela et améliorer un peu les performances, on passe `tcs` via le paramètre `state`.

On peut maintenant utiliser notre méthode `WaitOneAsync` de la façon suivante :

```
var mre = new ManualResetEvent(false);
…
if (await mre.WaitOneAsync(2000, cancellationToken))
{
    …
}
```

Remarque importante : cette méthode ne fonctionnera pas pour un `Mutex`, car elle repose sur `RegisterWaitForSingleObject`, qui d’après la documentation ne fonctionne qu’avec les wait handles autres que `Mutex`.

### Conclusion

On a donc vu qu’avec juste quelques méthodes d’extension, on peut rendre les primitives de synchronisation standard bien plus pratiques à utiliser dans du code moderne qui tire parti de l’asynchronisme et de l’annulation. Cependant je peux difficilement terminer ce billet sans mentionner la librairie [AsyncEx](https://github.com/StephenCleary/AsyncEx) de Stephen Cleary ; c’est une boite à outils très complète qui fournit des versions asynchrones de la plupart des primitives de synchronisation, dont certaines qui permettront d’arriver au même résultat que le code ci-dessus. Je vous invite à y jeter un œil, elle contient plein de choses utiles.

