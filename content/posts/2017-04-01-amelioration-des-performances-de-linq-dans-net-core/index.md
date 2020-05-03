---
layout: post
title: Amélioration des performances de Linq dans .NET Core
date: 2017-04-01T00:00:00.0000000
url: /2017/04/01/amelioration-des-performances-de-linq-dans-net-core/
tags:
  - .net core
  - C#
  - linq
  - performance
categories:
  - Uncategorized
---


Depuis le temps qu'on en parle, vous êtes sans doute au courant que Microsoft a publié une version open-source et multiplateforme de .NET : [.NET Core](https://github.com/dotnet/corefx). Cela signifie que vous pouvez maintenant créer et exécuter des applications .NET sous Linux ou macOS. C'est déjà assez cool en soi, mais ça ne s'arrête pas là : .NET Core apporte aussi beaucoup d'améliorations à la Base Class Library.

Par exemple, Linq est plus rapide dans .NET Core. J'ai fait un petit benchmark pour comparer les performances de certaines méthodes couramment utilisées de Linq, et les résultats sont assez impressionnants :
<!--![Performance comparison](perf.png)--><iframe width="700" height="210" frameborder="0" scrolling="no" src="https://onedrive.live.com/embed?cid=D2FB47CF02C0FD46&resid=D2FB47CF02C0FD46%21439375&authkey=AGAkuUFFLgMK5_Q&em=2&wdAllowInteractivity=False&ActiveCell='Sheet1'!A2&Item='Sheet1'!A1%3AG8&wdHideGridlines=True&wdDownloadButton=True"></iframe>
Le code complet de ce benchmark est disponible [ici](https://github.com/thomaslevesque/TestLinqPerf). Comme pour tous les microbenchmarks, les résultats ne sont pas à prendre pour argent comptant, mais ça donne quand même une idée des améliorations.

Certaines lignes de ce tableau sont assez surprenantes. Comment `Select` peut-il s'exécuter 5000 fois presque instantanément ? D'abord, il faut garder à l'esprit que la plupart des opérateurs Linq ont une exécution différée : ils ne font rien tant que qu'on n'énumère pas le résultat, donc quelque chose comme `array.Select(i => i * i)` s'exécute en temps constant (ça renvoie juste une séquence "lazy", sans consommer les éléments de `array`). C'est pourquoi j'ai ajouté un appel à `Count()` dans mon benchmark, pour m'assurer que le résultat est bien énuméré.

Pourtant, ce test s'exécute 5000 fois en 413µs... Cela est possible grâce à une optimisation dans l'implémentation .NET Core de `Select` et `Count`. Une propriété utile de `Select` est qu'il produit une séquence avec le même nombre d'éléments que la séquence source. Dans .NET Core, `Select` tire parti de cette propriété. Si la source est une `ICollection<T>` ou un tableau, il renvoie un objet énumérable qui garde la trace du nombre d'élément. `Count` peut ensuite récupérer directement la valeur et la renvoyer sans énumérer la séquence, ce qui donne un résultat en temps constant. L'implémentation de .NET 4.6.2, en revanche, énumère naïvement la séquence produite par `Select`, ce qui prend beaucoup plus longtemps.

Il est intéressant de noter que dans cette situation, .NET Core ne va *pas* exécuter la projection spécifiée dans `Select`, c'est donc un breaking change par rapport à .NET 4.6.2 pour du code qui dépend des effets de bord de la projection. Cela a été identifié comme un [problème](https://github.com/dotnet/corefx/pull/14435), qui a déjà été corrigé sur la branche master, donc la prochaine version n'aura plus cette optimisation et exécutera bien la projection sur chaque élément.

`OrderBy` suivi de `Count()` s'exécute aussi presque instantanément... Les développeurs de Microsoft auraient-ils inventé un algorithme de tri en `O(1)` ? Malheureusement, non... L'explication est la même que pour `Select` : puisque `OrderBy` préserve le nombre d'éléments, l'information est conservée pour pouvoir être utilisée par `Count`, et il n'est pas nécessaire de vraiment trier les éléments avant d'obtenir leur nombre.

Bon, ces cas étaient des améliorations assez évidentes (qui ne vont d'ailleurs pas rester, comment mentionné précédemment). Mais que dire du cas de `SelectAndToArray` ? Dans ce test, j'appelle `ToArray()` sur le résultat de `Select`, pour être certain que la projection soit bien exécutée sur chaque élément : cette fois, on ne triche pas. Pourtant, l'implémentation de .NET Core arrive encore à être 68% plus rapide que celle du framework .NET classique dans ce scénario. La raison est liée aux allocations : puisque l'implémentation .NET Core sait combien il y a d'éléments dans le résultat de `Select`, elle peut directement allouer un tableau de la bonne taille. Dans .NET 4.6.2, cette information n'est pas disponible, donc il faut commencer par allouer un petit tableau, y copier des éléments jusqu'à ce qu'il soit plein, puis allouer un tableau plus grand, y copier les données du premier tableau, y copier les éléments suivants de la séquence jusqu'à ce qu'il soit plein, etc. Cela cause de nombreuses allocations et copies, d'où la performance dégradée. Il y a quelques années, j'avais [suggéré](https://www.thomaslevesque.fr/2014/12/08/optimiser-toarray-et-tolist-en-fournissant-le-nombre-dlments/) des versions optimisées de `ToList` et `ToArray`, auxquelles on passait le nombre d'éléments. L'implémentation de .NET Core fait grosso modo la même chose, sauf qu'il n'y a pas besoin de passer la taille manuellement, elle est transmise à travers la chaine de méthodes Linq.

`Where` et `WhereAndToArray` sont tous les deux environ 8% plus rapides sur .NET Core 1.1. En examinant le code, ([.NET 4.6.2](https://referencesource.microsoft.com/#System.Core/System/Linq/Enumerable.cs,ed14299f42af7eb2), [.NET Core](https://github.com/dotnet/corefx/blob/e5cba5572d5b3634e768e3df3ddb5399fcf969b1/src/System.Linq/src/System/Linq/Where.cs#L208)), je ne vois pas de différences évidentes qui pourraient expliquer les meilleures performances, donc je soupçonne qu'elles sont dues à des améliorations du runtime. Dans ce cas, `ToArray` ne connait pas le nombre d'éléments dans la séquence (puisqu'on ne peut pas prédire le nombre d'éléments que `Where` va laisser passer), il ne peut donc pas utiliser la même optimisation qu'avec `Select`, et doit construire le tableau en utilisant l'approche plus lente.

On a déjà parlé du cas de `OrderBy` + `Count()`, qui n'était pas une comparaison équitable puisque l'implémentation de .NET Core ne triait pas réellement la séquence. Le cas de `OrderByAndToArray` est plus intéressant, car le tri ne peut pas être évité. Et dans ce cas, l'implémentation de .NET Core est un peu *plus lente* que celle de .NET 4.6.2. Je ne sais pas très bien pourquoi; là aussi, l'implémentation est très similaire, à part quelques refactorisations dans la version .NET Core.

Au final, Linq a l'air globalement plus rapide dans .NET Core que dans 4.6.2, ce qui est une très bonne nouvelle. Bien sûr, je n'ai benchmarké qu'un nombre limité de scénarios, mais cela montre quand même que l'équipe .NET Core travaille dur pour optimiser tout ce qu'ils peuvent.

