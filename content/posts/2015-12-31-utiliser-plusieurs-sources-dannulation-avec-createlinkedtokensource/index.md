---
layout: post
title: Utiliser plusieurs sources d’annulation avec CreateLinkedTokenSource
date: 2015-12-31T00:00:00.0000000
url: /2015/12/31/utiliser-plusieurs-sources-dannulation-avec-createlinkedtokensource/
tags:
  - annulation
  - async
  - C#
  - CancellationTokenSource
categories:
  - Uncategorized
---


La programmation asynchrone en C# était auparavant quelque chose de difficile ; grâce à la Task Parallel Library de .NET 4 et au async/await de C# 5, elle est devenu relativement facile, et en conséquence, est de plus en plus couramment utilisée. Dans le même temps, [une approche standardisée pour gérer l’annulation](https://msdn.microsoft.com/fr-fr/library/dd997364%28v=vs.110%29.aspx) a été introduite : les jetons d’annulation. L’idée générale est que vous créez un `CancellationTokenSource` qui contrôle l’annulation, et vous passez le jeton qu’il fournit à la méthode que vous voulez pouvoir annuler. Cette méthode le passera ensuite aux autres méthodes qu’elle appelle si elles peuvent être annulées, et/où vérifiera régulièrement si l’annulation a été demandée. Lors de l’annulation, la méthode lancera une `OperationCanceledException`. Exemple vite fait mal fait :

```
private readonly IBusinessService _businessService;
private CancellationTokenSource _cancellationSource;
private Task _asyncOperation;
 
private void StartAsyncOperation()
{
    if (_asyncOperation != null)
        return;
    var _cancellationSource = new CancellationTokenSource();
    _asyncOperation = _businessService.DoSomethingAsync(_cancellationSource.Token);
}
 
// async void is bad; like I said, this is a quick and dirty example
private async void StopAsyncOperation()
{
    try
    {
        _cancellationSource.Cancel();
        // wait for the operation to finish
        await _asyncOperation;
    }
    catch (OperationCanceledException)
    {
        // Operation was successfully canceled
    }
    catch (Exception)
    {
        // Oops, something went wrong
    }
    finally
    {
        _asyncOperation = null;
        _cancellationSource.Dispose();
        _cancellationSource = null;
    }
 
...
 
class BusinessService : IBusinessService
{
    public async Task DoSomethingAsync(CancellationToken cancellationToken)
    {
        var data = await GetDataFromServerAsync(cancellationToken);
        foreach (string line in data)
        {
            cancellationToken.ThrowIfCancellationRequested();
            await ProcessLineAsync(line, cancellationToken);
        }
    }
 
    ...
}
```

Dans ce cas, `StopAsyncOperation` serait appelée, par exemple, si l’utilisateur décide d’annuler l’opération.

Tout ça marche très bien et est assez facile à mettre en œuvre. Mais que se passe-t-il s’il y a une autre raison d’annuler l’opération, connue seulement du `BusinessService` et hors du contrôle de la méthode appelante ? C’est là que la méthode `CancellationSource.CreateLinkedTokenSource` entre en jeu ; en gros, cette méthode crée une nouvelle source d’annulation qui sera annulée quand l’un des jetons spécifiés sera annulé.

Commençons par un cas simple : vous avez un autre jeton d’annulation que vous voulez aussi prendre en compte. Le code pourrait ressembler à ceci :

```
public async Task DoSomethingAsync(CancellationToken cancellationToken)
{
    var otherToken = GetOtherCancellationToken();
    using (var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken, otherToken))
    {
        var data = await GetDataFromServerAsync(linkedCts.Token);
        foreach (string line in data)
        {
            linkedCts.Token.ThrowIfCancellationRequested();
            await ProcessLineAsync(line, linkedCts.Token);
        }
    }
}
```

On crée une source d’annulation liée, basée sur les deux jetons d’annulation, et on utilise le jeton de cette nouvelle source à la place de `cancellationToken`. Si `cancellationToken` ou `otherToken` est annulé, `linkedCts.Token` sera annulé aussi. Si nécessaire, le code appelant peut détecter comment l’opération a été annulée en vérifiant la propriété `CancellationToken` de l’exception `OperationCanceledException`.

Maintenant, un cas un peu plus difficile : la seconde source d’annulation est en fait un évènement. On veut annuler l’opération quand cet évènement se produit, en plus de l’annulation par l’utilisateur représentée par le paramètre `cancellationToken`. On va donc s’abonner à l’évènement et déclencher l’annulation quand il se produit. Voilà un moyen de le faire :

```
public async Task DoSomethingAsync(CancellationToken cancellationToken)
{
    using (var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken))
    {
        EventHandler handler = (sender, e) => linkedCts.Cancel();
        try
        {
            SomeEvent += handler;
            var data = await GetDataFromServerAsync(linkedCts.Token);
            foreach (string line in data)
            {
                linkedCts.Token.ThrowIfCancellationRequested();
                await ProcessLineAsync(line, linkedCts.Token);
            }
        }
        finally
        {
            SomeEvent -= handler;
        }
    }
}
```

Ici on passe seulement `cancellationToken` à `CreateLinkedTokenSource`, et on annule directement `linkedCts` quand l’évènement se produit. Le code devient un peu plus alambiqué, mais il atteint l’objectif.

Je ne peux pas vraiment vous donner un cas d’utilisation spécifique de cette technique dans le monde réel, parce que les cas où je l’ai utilisée sont trop spécifiques pour être d’intérêt général, mais je peux décrire le scénario général. J’ai une opération de longue durée qui est constituée de plusieurs sous-opérations de longue durée. L’opération entière peut être annulée globalement, et chacune des sous-opérations peut également être annulée individuellement, sans affecter les autres. Voilà à quoi ça peut ressembler (en quasi pseudo-code) :

```
async Task GlobalOperationAsync(CancellationToken cancellationToken)
{
    foreach (var subOperation is SubOperations)
    {
        cancellationToken.ThrowIfCancellationRequested();
        var subToken = subOperation.GetSpecificCancellationToken();
        using (var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken, subToken))
        {
            try
            {
                await subOperation.RunAsync(linkedCts.Token);
            }
            catch (OperationCanceledException ex)
            {
                // Rethrow only if global cancellation was requested
                if (cancellationToken.IsCancellationRequested)
                    throw;
                     
                // otherwise continue running the other sub-operations
            }
        }
    }
}
```

Remarquez que, bien que `CancellationToken` ait été introduit avec la TPL et que tous mes exemples soient asynchrones, vous pouvez également utiliser cette technique avec du code synchrone.

Voilà, j’espère que cela vous sera utile. Passez un bon réveillon du nouvel an, et une excellente année 2016 !

