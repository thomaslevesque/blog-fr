---
layout: post
title: Optimiser ToArray et ToList en fournissant le nombre d’éléments
date: 2014-12-08T00:00:00.0000000
url: /2014/12/08/optimiser-toarray-et-tolist-en-fournissant-le-nombre-dlments/
tags:
  - C#
  - linq
  - optimisation
  - performance
  - ToArray
  - ToList
categories:
  - Uncategorized
---


Les méthodes d’extension `ToArray` et `ToList` sont des moyens pratiques de matérialiser une séquence énumérable (par exemple une requête Linq). Cependant, quelque chose me chiffonne : ces deux méthodes sont très inefficaces si elles ne connaissent pas le nombre d’éléments dans la séquence (ce qui est presque toujours le cas quand on les utilise sur une requête Linq). Concentrons nous sur `ToArray` pour l’instant (`ToList` a quelques différences, mais le principe est essentiellement le même).

Pour faire simple, `ToArray` prend une séquence, et retourne un tableau qui contient tous les éléments de la séquence. Si la séquence implémente `ICollection<T>`, `ToArray` utilise la propriété `Count` pour allouer un tableau de la bonne taille, et copie les éléments dedans ; voici un exemple :



```
List<User> users = GetUsers();
User[] array = users.ToArray();
```

Dans ce scénario, `ToArray` est assez efficace. Maintenant, changeons ce code pour extraire les noms des utilisateurs :



```
List<User> users = GetUsers();
string[] array = users.Select(u => u.Name).ToArray();
```

Maintenant, l’argument de `ToArray` est un `IEnumerable<User>` renvoyé par `Select`. Il n’implémente pas `ICollection<User>`, donc `ToArray` ne connait pas le nombre d’éléments, et ne peut donc pas allouer un tableau de la bonne taille. Voilà donc ce qu’il fait :

1. commencer par allouer un petit tableau (4 éléments dans [l’implémentation actuelle](http://referencesource.microsoft.com/#System.Core/System/Linq/Enumerable.cs,783a052330e7d48d))
2. copier les éléments depuis la source vers le tableau jusqu’à ce que celui-ci soit plein
3. s’il n’y a plus d’éléments dans la séquence, aller en 7
4. sinon, allouer un nouveau tableau deux fois plus grand que le précédent
5. copier les éléments de l’ancien tableau vers le nouveau
6. répéter depuis l’étape 2
7. si le tableau est plus long que le nombre d’éléments, le tronquer : allouer un nouveau tableau d’exactement la bonne taille, et y copier les éléments depuis le tableau précédent
8. renvoyer le tableau


S’il n’y a que quelques éléments, tout ceci est assez indolore ; mais pour une très longue séquence, c’est très inefficace, à cause des nombreuses allocations et copies.

Ce qui est agaçant est que, bien souvent, on (le développeur) *connait* le nombre d’éléments de la source! Dans l’exemple ci-dessus, on n’utilise que `Select`, qui ne change pas le nombre d’éléments, donc on sait que c’est le même que dans la liste d’origine; mais `ToArray` ne le sait pas, car l’information a été perdue en cours de route. Si seulement on pouvait fournir cette information nous-mêmes…

Eh bien, c’est en fait très facile à faire : il suffit de créer une nouvelle méthode d’extension qui accepte le nombre d’éléments comme paramètre. Voilà à quoi elle pourrait ressembler :



```
public static TSource[] ToArray<TSource>(this IEnumerable<TSource> source, int count)
{
    if (source == null) throw new ArgumentNullException("source");
    if (count < 0) throw new ArgumentOutOfRangeException("count");
    var array = new TSource[count];
    int i = 0;
    foreach (var item in source)
    {
        array[i++] = item;
    }
    return array;
}
```

On peut maintenant optimiser notre exemple précédent de la façon suivante :



```
List<User> users = GetUsers();
string[] array = users.Select(u => u.Name).ToArray(users.Count);
```

Notez que si vous spécifiez un nombre inférieur au nombre réel d’éléments dans la séquence, vous obtiendrez une `IndexOutOfRangeException` ; il est de votre responsabilité de fournir une information correcte à la méthode.

Et au final, qu’est-ce qu’on y gagne? D’après mes tests, cette version améliorée de `ToArray` est environ **deux fois plus rapide** que la version standard, pour une longue séquence (testé avec 1.000.000 éléments). Pas mal du tout !

Notez qu’on peut améliorer `ToList` de la même manière, en utilisant le constructeur de `List<T>` qui permet de spécifier la capacité initiale :



```
public static List<TSource> ToList<TSource>(this IEnumerable<TSource> source, int count)
{
    if (source == null) throw new ArgumentNullException("source");
    if (count < 0) throw new ArgumentOutOfRangeException("count");
    var list = new List<TSource>(count);
    foreach (var item in source)
    {
        list.Add(item);
    }
    return list;
}
```

Ici, le gain de performance n’est pas aussi important que pour `ToArray` (environ 25% au lieu de 50%), probablement parce que la liste n’a pas besoin d’être tronquée, mais il n’est pas négligeable.

Bien sûr, une optimisation similaire pourrait être faite pour `ToDictionary`, puisque la classe `Dictionary<TKey, TValue>` a aussi un constructeur qui permet de spécifier la capacité initiale.

Les méthodes `ToArray` et `ToList` améliorées sont disponibles dans ma librairie [Linq.Extras](https://github.com/thomaslevesque/Linq.Extras), qui fournit également de nombreuses méthodes d’extension utiles pour travailler avec des séquences et collections.

