---
layout: post
title: Meilleure gestion du timeout avec HttpClient
date: 2018-02-25T00:00:00.0000000
url: /2018/02/25/meilleure-gestion-du-timeout-avec-httpclient/
tags:
  - .NET
  - C#
  - handler
  - HTTP
  - HttpClient
  - timeout
categories:
  - Uncategorized
---


## Le problème

Si vous avez l'habitude d'utiliser `HttpClient` pour appeler des APIs REST ou transférer des fichiers, vous avez peut-être déjà pesté contre la façon dont cette classe gère le **timeout**. Il y a en effet deux problèmes majeurs dans la gestion du timeout par `HttpClient` :

- **Le timeout est défini de façon globale**, et s'applique à *toutes* les requêtes, alors qu'il serait plus pratique de pouvoir le définir individuellement pour chaque requête.
- L'exception levée quand le temps imparti est écoulé **ne permet pas de déterminer la cause de l'erreur**. En effet, en cas de timeout, on s'attendrait à recevoir une `TimeoutException`, non ? Eh bien, surprise, c'est une `TaskCanceledException` qui est levée! Du coup, impossible de savoir si la requête a réellement été annulée, ou si le timeout est écoulé.


Heureusement, tout n'est pas perdu, et la flexibilité de `HttpClient` va permettre de compenser cette petite erreur de conception...

On va donc implémenter un mécanisme permettant de pallier les deux problèmes mentionnés plus haut. On souhaite donc :

- pouvoir **spécifier un timeout différent pour chaque requête**
- **recevoir une `TimeoutException` plutôt que `TaskCanceledException`** en cas de timeout


## Spécifier le timeout pour une requête

Voyons d'abord comment associer une valeur de timeout à une requête. La classe `HttpRequestMessage` a une propriété `Properties`, qui est un dictionnaire dans lequel on peut mettre ce qu'on veut. On va donc l'utiliser pour stocker le timeout pour une requête, et pour faciliter les choses, on va créer des méthodes d'extension pour accéder à la valeur de façon fortement typée :

```csharp
public static class HttpRequestExtensions
{
    private static string TimeoutPropertyKey = "RequestTimeout";

    public static void SetTimeout(
        this HttpRequestMessage request,
        TimeSpan? timeout)
    {
        if (request == null)
            throw new ArgumentNullException(nameof(request));

        request.Properties[TimeoutPropertyKey] = timeout;
    }

    public static TimeSpan? GetTimeout(this HttpRequestMessage request)
    {
        if (request == null)
            throw new ArgumentNullException(nameof(request));

        if (request.Properties.TryGetValue(
                TimeoutPropertyKey,
                out var value)
            && value is TimeSpan timeout)
            return timeout;
        return null;
    }
}
```

Rien de très compliqué ici, le timeout est une valeur optionnelle de type `TimeSpan`. Évidemment il n'y a pour l'instant aucun code pour tenir compte du timeout associé à une requête...

## Handler HTTP

L'architecture de `HttpClient` est basée sur **un système de pipeline** : chaque requête est envoyée à travers une chaîne de handlers (de type `HttpMessageHandler`), et la réponse repasse en sens inverse à travers cette chaîne. [Cet article](https://www.thomaslevesque.fr/2016/12/11/tout-faire-ou-presque-avec-le-pipeline-de-httpclient/) rentre un peu plus dans le détail si vous voulez en savoir plus. Nous allons donc **insérer dans le pipeline notre propre handler**, qui sera chargé de la gestion du timeout.

Notre handler va hériter de `DelegatingHandler`, un type de handler conçu pour être chaîné à un autre handler. Pour implémenter un handler, il faut **redéfinir la méthode `SendAsync`**. Une implémentation minimale ressemblerait à ceci :

```csharp
class TimeoutHandler : DelegatingHandler
{
    protected async override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        return await base.SendAsync(request, cancellationToken);
    }
}
```

L'appel à `base.SendAsync` va simplement passer la requête au handler suivant. Du coup, pour l'instant notre implémentation ne sert à rien, mais on va l'enrichir petit à petit.

## Prendre en compte le timeout pour une requête

Ajoutons d'abord à notre classe une propriété `DefaultTimeout`, qui sera utilisée pour les requêtes dont le timeout n'est pas explicitement défini :

```csharp
public TimeSpan DefaultTimeout { get; set; } = TimeSpan.FromSeconds(100);
```

La valeur par défaut de 100 secondes est la même que celle de `HttpClient.Timeout`.

Pour implémenter le timeout, on va récupérer la valeur associée à la requête (ou à défaut `DefaultTimeout`), **créer un `CancellationToken` qui sera annulé après la durée du timeout, et passer ce `CancellationToken` au handler suivant** : la requête sera donc annulée après l'expiration de ce délai (ce qui correspond au comportement par défaut de `HttpClient`).

Pour créer un `CancellationToken` dont on peut contrôler l'annulation, **on utilise un objet `CancellationTokenSource`**, qu'on va créer comme ceci en fonction du timeout de la requête :

```csharp
private CancellationTokenSource GetCancellationTokenSource(
    HttpRequestMessage request,
    CancellationToken cancellationToken)
{
    var timeout = request.GetTimeout() ?? DefaultTimeout;
    if (timeout == Timeout.InfiniteTimeSpan)
    {
        // No need to create a CTS if there's no timeout
        return null;
    }
    else
    {
        var cts = CancellationTokenSource
            .CreateLinkedTokenSource(cancellationToken);
        cts.CancelAfter(timeout);
        return cts;
    }
}
```

Deux choses à noter ici :

- si le timeout de la requête est infini, on ne crée pas de `CancellationTokenSource`; il ne servirait à rien puisqu'il ne serait jamais annulé, on économise donc une allocation inutile.
- Dans le cas contraire, on crée un `CancellationTokenSource` qui sera annulé après expiration du timeout (`CancelAfter`). Notez que **ce CTS est *lié au `CancellationToken` reçu en paramètre de `SendAsync`***: il sera donc annulé soit par après expiration du timeout, soit quand ce `CancellationToken` sera lui-même annulé. Je vous renvoie à [cet article](https://www.thomaslevesque.fr/2015/12/31/utiliser-plusieurs-sources-dannulation-avec-createlinkedtokensource/) pour plus d'infos à ce sujet.


Enfin, modifions la méthode `SendAsync` pour prendre en compte le `CancellationTokenSource` qu'on a créé :

```csharp
protected async override Task<HttpResponseMessage> SendAsync(
    HttpRequestMessage request,
    CancellationToken cancellationToken)
{
    using (var cts = GetCancellationTokenSource(request, cancellationToken))
    {
        return await base.SendAsync(
            request,
            cts?.Token ?? cancellationToken);
    }
}
```

On récupère le CTS, et on passe son token à `base.SendAsync`. Notez qu'on utilise `cts?.Token` puisque `GetCancellationTokenSource` peut renvoyer null; si c'est le cas, on utilise le `CancellationToken` reçu en paramètre.

À ce stade, on a un handler qui permet de spécifier un timeout différent pour chaque requête. Mais il reste le problème de l'exception renvoyée en cas de timeout, qui est encore une `TaskCanceledException`... Mais on va régler ça très facilement!

## Lever la bonne exception

En effet, il suffit d'intercepter l'exception `TaskCanceledException` (ou plutôt sa classe de base, `OperationCanceledException`), et de **vérifier si le `CancellationToken` reçu en paramètre est annulé**: si oui, l'annulation vient de l'appelant, et on laisse l'exception se propager normalement; si non, c'est qu'elle est causée par le timeout, et dans ce cas on lance une `TimeoutException`. Voilà donc notre méthode `SendAsync` finale:

```csharp
protected async override Task<HttpResponseMessage> SendAsync(
    HttpRequestMessage request,
    CancellationToken cancellationToken)
{
    using (var cts = GetCancellationTokenSource(request, cancellationToken))
    {
        try
        {
            return await base.SendAsync(
                request,
                cts?.Token ?? cancellationToken);
        }
        catch(OperationCanceledException)
            when (!cancellationToken.IsCancellationRequested)
        {
            throw new TimeoutException();
        }
    }
}
```

On utilise ici un [filtre d'exception](https://www.thomaslevesque.fr/2015/06/23/filtres-dexception-en-c-6-leur-plus-grand-avantage-nest-pas-celui-quon-croit/) : cela évite d'intercepter `OperationCanceledException` si on doit la laisser se propager; on évite ainsi de dérouler la pile inutilement.

Notre handler est terminé, voyons maintenant comment l'utiliser.

## Utilisation du handler

Quand on crée un `HttpClient`, il est possible de **passer en paramètre du constructeur le premier handler du pipeline**. Si on ne spécifie rien, par défaut c'est un `HttpClientHandler` qui est créé; ce handler envoie directement les requêtes vers le réseau. Pour utiliser notre nouveau `TimeoutHandler`, on va le créer, lui attacher un `HttpClientHandler` comme handler suivant, et le passer au `HttpClient`:

```csharp
var handler = new TimeoutHandler
{
    InnerHandler = new HttpClientHandler()
};

using (var client = new HttpClient(handler))
{
    client.Timeout = Timeout.InfiniteTimeSpan;
    ...
}
```

Notez qu'il faut **désactiver le timeout du `HttpClient` en lui donnant une valeur infinie**, sinon le comportement par défaut viendra interférer avec notre handler.

Essayons maintenant d'envoyer une requête avec un timeout de 5 secondes vers un serveur qui met trop longtemps à répondre:

```csharp
var request = new HttpRequestMessage(HttpMethod.Get, "http://foo/");
request.SetTimeout(TimeSpan.FromSeconds(5));
var response = await client.SendAsync(request);
```

Si le serveur n'a pas répondu au bout de 5 secondes, on obtiendra bien une `TimeoutException`, et non une `TaskCanceledException`.

Vérifions maintenant que le cas de l'annulation marche toujours correctement. Pour cela, on va passer un `CancellationToken` qui sera annulé au bout de 2 secondes (avant expiration du timeout, donc) :

```csharp
var request = new HttpRequestMessage(HttpMethod.Get, "http://foo/");
request.SetTimeout(TimeSpan.FromSeconds(5));
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(2));
var response = await client.SendAsync(request, cts.Token);
```

Et on obtient bien une `TaskCanceledException`!

En implémentant notre propre handler HTTP, on a donc pu régler notre problème de départ et avoir une gestion intelligente du timeout.

Le code complet de cet article est disponible [ici](https://gist.github.com/thomaslevesque/b4fd8c3aa332c9582a57935d6ed3406f).

