---
layout: post
title: Déconstruction de tuples en C# 7
date: 2016-08-23T00:00:00.0000000
url: /2016/08/23/deconstruction-de-tuples-en-c-7/
tags:
  - C#
  - C# 7
  - deconstruction
  - tuple
  - Visual Studio 15
categories:
  - Uncategorized
---


Dans mon [précedent billet](http://www.thomaslevesque.fr/2016/07/28/tuples-en-c-7/), j'ai parlé d'une nouvelle fonctionnalité de C# 7 : les tuples. Dans Visual Studio 15 Preview 3, cette feature n'était pas tout à fait terminée ; il lui manquait 2 aspects importants :

- la génération de métadonnées pour les noms des éléments des tuples, pour que les noms soient préservés entre les assemblies
- la déconstruction des tuples en variables distinctes


Eh bien, il semble que l'équipe du langage C# n'a pas chômé au cours du mois écoulé, car ces deux éléments sont maintenant implémentés dans VS 15 Preview 4, qui a été [publié hier](https://blogs.msdn.microsoft.com/visualstudio/2016/08/22/visual-studio-15-preview-4/) ! Ils ont aussi rédigé des guides sur l'utilisation des [tuples](https://github.com/dotnet/roslyn/blob/master/docs/features/tuples.md) et de la [déconstruction](https://github.com/dotnet/roslyn/blob/master/docs/features/deconstruction.md).

Il est maintenant possible d'écrire des choses comme ça :

```csharp
var values = ...
var (count, sum) = Tally(values);
Console.WriteLine($"There are {count} values and their sum is {sum}");
```

(la méthode `Tally` est celle du précédent billet)

Remarquez que la variable `t` du billet précédent a disparu ; on affecte maintenant directement les variables `count` et `sum` à partir du résultat de la méthode, ce qui est à mon sens beaucoup plus élégant. Il ne semble pas y avoir de moyen d'ignorer une partie du tuple (c'est-à-dire ne pas l'affecter à une variable), mais peut-être cette possibilité viendra-t-elle plus tard.

Un aspect intéressant de la déconstruction est qu'elle n'est pas limitée aux tuples ; n'importe quel type peut être déconstruit, à condition d'avoir une méthode `Deconstruct` avec les paramètres `out` adéquats :

```csharp
class Point
{
    public int X { get; }
    public int Y { get; }

    public Point(int x, int y)
    {
        X = x;
        Y = y;
    }

    public void Deconstruct(out int x, out int y)
    {
        x = X;
        y = Y;
    }
}

...

var (x, y) = point;
Console.WriteLine($"Coordinates: ({x}, {y})");
```

La méthode `Deconstruct` peut également être une méthode d'extension, ce qui peut être utile si vous voulez déconstruire un type dont vous ne contrôlez pas le code. Les vieilles classes `Sytem.Tuple`, par exemple, peuvent être déconstruites à l'aide de méthodes d'extension comme celle-ci :

```csharp
public static void Deconstruct<T1, T2>(this Tuple<T1, T2> tuple, out T1 item1, out T2 item2)
{
    item1 = tuple.Item1;
    item2 = tuple.Item2;
}

...

var tuple = Tuple.Create("foo", 42);
var (name, value) = tuple;
Console.WriteLine($"Name: {name}, Value = {value}");
```

Pour finir, les méthodes qui renvoient des tuples sont maintenant décorées d'un attribut `[TupleElementNames]` qui indique les noms des éléments du tuple :

```csharp
// Code décompilé
[return: TupleElementNames(new[] { "count", "sum" })]
public static ValueTuple<int, double> Tally(IEnumerable<double> values)
{
   ...
}
```

(l'attribut est généré par le compilateur, vous n'avez pas besoin de l'écrire vous-même)

Cela permet de partager les noms des éléments du tuple entre les assemblies, et permet aux outils comme Intellisense de fournir des informations utiles sur la méthode.

L'implémentation des tuples en C# 7 semble donc à peu près terminée ; gardez cependant à l'esprit qu'il s'agit encore d'une preview, et que les choses peuvent encore changer d'ici la release finale.

