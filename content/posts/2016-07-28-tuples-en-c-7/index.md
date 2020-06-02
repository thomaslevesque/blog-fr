---
layout: post
title: Tuples en C# 7
date: 2016-07-28T00:00:00.0000000
url: /2016/07/28/tuples-en-c-7/
tags:
  - C#
  - C# 7
  - tuple
  - Visual Studio
categories:
  - Uncategorized
---


Un tuple est une liste finie et ordonnée de valeurs, éventuellement de types différents, et est utilisé pour regrouper des valeurs liées entre elles sans avoir à créer une type spécifique pour les contenir.

.NET 4.0 a introduit un ensemble de classes `Tuple` , qui s’utilisent de la façon suivante

```csharp
private static Tuple<int, double> Tally(IEnumerable<double> values)
{
    int count = 0;
    double sum = 0.0;
    foreach (var value in values)
    {
        count++;
        sum += value;
    }
    return Tuple.Create(count, sum);
}

...

var values = ...
var t = Tally(values);
Console.WriteLine($"There are {t.Item1} values and their sum is {t.Item2}");
```

Les classes `Tuple` ont deux principaux inconvénients:

- Ce sont des classes, c’est-à-dire des types référence. Cela implique qu’elles doivent être allouées sur le tas, et collectées par le garbage collector quand elles ne sont plus utilisées. Pour les applications où les performances sont critiques, cela peut être problématique. De plus, le fait qu’elles puissent être nulles n’est souvent pas souhaitable.
- Les éléments du tuple n’ont pas de noms, ou plutôt, ils ont toujours les mêmes noms (`Item1`, `Item2`, etc), qui ne sont pas du tout significatifs. Le type `Tuple<T1, T2>` ne communique absolument aucune information sur ce que le tuple représente réellement, ce qui en fait un mauvais candidat pour les APIs publiques.


En C# 7, une nouvelle fonctionnalité sera introduite pour améliorer le support des tuples : il sera possible de déclarer des types tuple “inline”, un peu comme des types anonymes, à part qu’ils ne sont pas limités à la méthode courante. En utilisant cette feature, le code précédent devient beaucoup plus clair:

```csharp
static (int count, double sum) Tally(IEnumerable<double> values)
{
    int count = 0;
    double sum = 0.0;
    foreach (var value in values)
    {
        count++;
        sum += value;
    }
    return (count, sum);
}

...

var values = ...
var t = Tally(values);
Console.WriteLine($"There are {t.count} values and their sum is {t.sum}");
```

Notez comment le type de retour de `Tally` est déclaré, et comment le résultat est utilisé. C’est beaucoup mieux ! Les éléments du tuple ont maintenant des noms significatifs, et la syntaxe est plus agréable. Cette fonctionnalité repose sur un nouveau type `ValueTuple<T1, T2>`, qui est une structure et ne nécessite donc pas d’allocation sur le tas.

Vous pouvez essayer cette feature dès maintenant dans Visual Studio 15 Preview 3. Cependant, le type `ValueTuple<T1, T2>` ne fait pas (encore) partie du .NET Framework; pour faire fonctionner cet exemple, il faudra installer le package NuGet [System.ValueTuple](https://packages.nuget.org/packages/System.ValueTuple).

Enfin, une dernière remarque concernant les noms des membres du tuple : comme beaucoup d’autres fonctionnalités du langage, c’est juste du sucre syntaxique. Dans le code compilé, les membres du tuple sont référencés en tant que `Item1` et `Item2`, et non `count` et `sum`. La méthode `Tally` renvoie en fait un `ValueTuple<int, double>`, et non un type spécialement généré.

Notez que le compilateur qui est livré avec VS 15 Preview 3 ne génère aucune métadonnée concernant les noms des membres du tuple. Cette partie de la feature n’est pas encore implémentée, mais devrait être incluse dans la version finale. Cela signifie qu’en attendant, on ne peut pas utiliser de tuples entre différents assemblies (enfin, on peut, mais en perdant les noms des membres, il faudra donc utiliser `Item1` et `Item2` pour y faire référence).

