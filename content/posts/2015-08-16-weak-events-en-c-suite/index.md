---
layout: post
title: Weak events en C#, suite
date: 2015-08-16T00:00:00.0000000
url: /2015/08/16/weak-events-en-c-suite/
tags:
  - évènements
  - fuite mémoire
  - open-instance delegate
  - weak event
categories:
  - Librairies
---


Il y a quelques années, j’ai blogué à propos d’une [implémentation générique du pattern “weak event” en C#](http://www.thomaslevesque.fr/2010/05/16/c-une-implementation-du-pattern-weakevent/). Le but était de pallier les problèmes de fuites mémoire liés aux évènements quand on oublie de s’en désabonner. L’implémentation était basée sur l’utilisation de références faibles sur les abonnés, de façon à éviter d’empêcher qu’ils soient libérés par le garbage collector.

Ma solution initiale était plus une preuve de concept qu’autre chose, et avait un sérieux problème de performance, dû à l’utilisation de `DynamicInvoke` à chaque fois que l’évènement était déclenché. Au fil des années, j’ai revisité le problème des “weak events” plusieurs fois, en apportant quelques améliorations à chaque fois, et j’ai maintenant une implémentation qui devrait être suffisamment performante pour la plupart des cas d’utilisation. L’API publique est similaire à celle de ma première solution. En gros, au lieu d’écrire un évènement comme ceci :

```
public event EventHandler MyEvent;
```

On l’écrit comme ceci :

```
private readonly WeakEventSource _myEventSource = new WeakEventSource();
public event EventHandler MyEvent
{
    add { _myEventSource.Subscribe(value); }
    remove { _myEventSource.Unsubscribe(value); }
}
```

Du point de vue de celui qui s’abonne à l’évènement, c’est exactement pareil qu’un évènement normal, mais l’abonné restera éligible à la garbage collection s’il n’est plus référencé nulle part ailleurs.

L’objet qui publie l’évènement peut le déclencher comme ceci :

```
_myEventSource.Raise(this, e);
```

Il y a une petite limitation : la signature de l’évènement *doit* être `EventHandler<TEventArgs>` (avec ce que vous voulez comme `TEventArgs`, bien sûr). Ca ne peut pas être quelque chose comme `FooEventHandler`, ou un type de délégué custom. Je ne pense pas que ce soit un problème majeur, dans la mesure où une vaste majorité des évènements dans le monde .NET respecte le pattern recommandé `void (sender, args)`, et les delegates spécifiques comme `FooEventHandler` ont en fait la même signature que `EventHandler<FooEventArgs>`. J’avais d’abord essayé de supporter n’importe quel type de delegate, mais ça s’est avéré un peu trop compliqué… pour l’instant en tout cas ![Winking smile](wlEmoticon-winkingsmile.png).



### Comment ça marche?

La nouvelle solution est encore basée sur des références faibles, mais change la façon dont la méthode cible est appelée. Au lieu d’utiliser `DynamicInvoke`, on crée un “open-instance delegate” pour la méthode lors de l’abonnement. Cela signifie que pour une méthode ayant une signature comme `void EventHandler(object sender, EventArgs e)`, on  crée un delegate avec la signature `void OpenEventHandler(object target, object sender, EventArgs e)`. Le paramètre supplémentaire `target` représente l’instance sur laquelle la méthode est appelée. Pour invoquer le gestionnaire de l’évènement, il suffit de récupérer la cible à partir de la référence faible, et si elle est toujours vivante, de la passer au “open-instance delegate”.

Pour de meilleures performances, ce delegate est en fait créé seulement la première fois qu’on rencontre une méthode donnée, et est mis en cache pour être réutilisé ultérieurement. Ainsi, si plusieurs instances d’une classe s’abonnent à l’évènement avec la même méthode, le delegate ne sera créé que la première fois, et sera réutilisé pour les abonnés suivants.

Notez que techniquement, le delegate créé n’est pas un “vrai” open-instance delegate comme ceux créés par la méthode `Delegate.CreateDelegate`. Il est en fait créé à l’aide des expressions Linq. La raison est que dans un vrai open-instance delegate, le type du premier paramètre doit être le type qui déclare la méthode, et non `object`. Puisque cette information n’est pas disponible statiquement, il faut introduire un cast dynamiquement.



Le code source est disponible sur GitHub: [WeakEvent](https://github.com/thomaslevesque/WeakEvent). Un package NuGet est disponible ici : [ThomasLevesque.WeakEvent](https://www.nuget.org/packages/ThomasLevesque.WeakEvent/).

Le dépôt GitHub contient aussi des snippets pour [Visual Studio](https://github.com/thomaslevesque/WeakEvent/blob/master/Snippets/VisualStudio/wevt.snippet) et pour [ReSharper](https://github.com/thomaslevesque/WeakEvent/blob/master/Snippets/ReSharper/wevt.DotSettings), pour faciliter l’écriture du code de plomberie pour un weak event.

