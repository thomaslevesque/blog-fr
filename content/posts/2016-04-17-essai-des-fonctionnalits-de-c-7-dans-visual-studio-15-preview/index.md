---
layout: post
title: Essai des fonctionnalités de C# 7 dans Visual Studio “15” Preview
date: 2016-04-17T00:00:00.0000000
url: /2016/04/17/essai-des-fonctionnalits-de-c-7-dans-visual-studio-15-preview/
tags:
  - C#
  - C# 7
  - Visual Studio
categories:
  - Uncategorized
---


Il y a environ deux semaines, Microsoft a publié la première version préliminaire de la prochaine mouture de Visual Studio. Vous pourrez découvrir toutes les nouveautés qu’elle contient dans les [notes de version](https://www.visualstudio.com/en-us/news/vs15-preview-vs.aspx). Il y a quelques nouveautés vraiment sympa (j’aime particulièrement le nouvel “installeur léger”), mais le plus intéressant pour moi est que le compilateur C# livré avec inclut quelques unes des fonctionnalités prévues pour C# 7. Regardons ça de plus près !

### Activer les nouvelles fonctionnalités

Les nouvelles fonctionnalités ne sont pas activées par défaut. On peut les activer individuellement avec l’option `/feature:` en ligne de commande, mais le plus simple est de toutes les activer en ajoutant `__DEMO__` et `__DEMO_EXPERIMENTAL__` dans les symboles de compilation conditionnelle (Propriétés du projet, onglet *Build*).

### Fonctions locales

La plupart des langages fonctionnels permettent de déclarer des fonctions dans le corps d’autres fonctions. Il est maintenant possible de faire la même chose en C# 7 ! La syntaxe pour déclarer une méthode dans une autre est sans grande surprise :

```
long Factorial(int n)
{
    long Fact(int i, long acc)
    {
        return i == 0 ? acc : Fact(i - 1, acc * i);
    }
    return Fact(n, 1);
}
```

Ici, la méthode `Fact` est locale à la méthode `Factorial` (au cas où vous vous posez la question, c’est une implémentation avec récursion terminale de la factorielle — ce qui n’a pas beaucoup de sens, vu que C# ne supporte pas la récursion terminale, mais bon, c’est juste un exemple…).

Bien sûr, il était déjà possible de simuler une fonction locale à l’aide d’une expression lambda, mais cela avait plusieurs inconvénients :

- c’est moins lisible, car il faut déclarer explicitement le type du delegate ;
- c’est plus lent, à cause du coût de création d’une instance de delegate, et de l’appel via le delegate ;
- l’écriture d’expressions lambda récursives est peu intuitive.


Voici les principaux avantages des fonctions locales :

- quand une méthode est utilisée seulement comme auxiliaire d’une autre méthode, la rendre locale permet de rendre cette relation plus explicite;
- comme les lambdas, une fonction locale peut capturer les variables et paramètres de la méthode qui la contient;
- les fonctions locales supportent la récursion comme n’importe quelle autre méthode


Pour en savoir plus sur les fonctions locales, allez voir [sur le dépôt Github de Roslyn](https://github.com/dotnet/roslyn/issues/2930).

### Valeurs de retour et variables locales par référence

Il est possible, depuis la première version de C#, de passer des paramètres par référence, ce qui est conceptuellement similaire au fait de passer un pointeur dans un langage comme C. Jusqu’ici, cette possibilité était limitée aux paramètres, mais C# 7 l’étend aux valeurs de retour et aux variables locales. Voici un exemple :

```
static void TestRefReturn()
{
    var foo = new Foo();
    Console.WriteLine(foo); // 0, 0
     
    foo.GetByRef("x") = 42;
 
    ref int y = ref foo.GetByRef("y");
    y = 99;
 
    Console.WriteLine(foo); // 42, 99
}
 
class Foo
{
    private int x;
    private int y;
 
    public ref int GetByRef(string name)
    {
        if (name == "x")
            return ref x;
        if (name == "y")
            return ref y;
        throw new ArgumentException(nameof(name));
    }
 
    public override string ToString() => $"{x},{y}";
}
```

Examinons ce code d’un peu plus près.

- A la ligne 6, on dirait que j’affecte une valeur à une méthode. Qu’est-ce que ça peut bien vouloir dire ? Eh bien, la méthode `GetByRef` renvoie un champ de la classe `Foo` *par référence* (notez le type de retour `ref int`). Si je passe `"x"` en argument, elle renvoie le champ `x` par référence (et non une copie de la valeur du champ). Donc si j’affecte une valeur à cela, la valeur du champ `x` est également modifiée.
- A la ligne 8, au lieu de simplement affecter une valeur directement au champ renvoyé par `GetByRef`, je passe par une variable locale `y`. Cette variable partage maintenant le même emplacement mémoire que le champ `foo.y`. Si j’affecte une valeur à `y`, cela modifie donc aussi `foo.y`.


Notez qu’il est également possible de renvoyer par référence un emplacement dans un tableau :

```
private MyBigStruct[] array = new MyBigStruct[10];
private int current;
 
public ref MyBigStruct GetCurrentItem()
{
    return ref array[current];
}
```

Il est probable que la plupart des développeurs C# n’auront jamais besoin de cette fonctionnalité; elle est assez bas-niveau, et ce n’est pas le genre de chose dont on a habituellement besoin quand on écrit des applications métier. Cependant, elle est très utile pour du code dont les performances sont critiques : copier une structure de grande taille est assez couteux, donc avoir la possibilité de la renvoyer plutôt par référence peut représenter un gain de performance non négligeable.

Pour en savoir plus sur cette fonctionnalité, rendez-vous [sur Github](https://github.com/dotnet/roslyn/issues/118).

### Pattern matching

Le pattern matching (parfois traduit en “filtrage par motif”, mais je trouve cette traduction un peu bancale) est une fonctionnalité très courante dans les langages fonctionnels. C# 7 introduit certains aspects du pattern matching, sous forme d’extensions de l’opérateur `is`. Par exemple, quand on teste le type d’une variable, on peut maintenant introduire une nouvelle variable après le type, de façon à ce que cette variable reçoive la valeur de l’opérande de gauche, mais avec le type indiqué par l’opérande de droite (ce sera plus clair dans un instant avec un exemple).

Par exemple, si vous devez tester qu’une valeur est de type `DateTime`, puis faire quelque chose avec ce `DateTime`, il faut actuellement tester le type, puis convertir la valeur vers ce type :

```
object o = GetValue();
if (o is DateTime)
{
    var d = (DateTime)o;
    // Do something with d
}
```

En C# 7, on peut simplifier un peu :

```
object o = GetValue();
if (o is DateTime d)
{
    // Do something with d
}
```

`d` est maintenant déclarée directement dans l’expression `o is DateTime`.

Cette fonctionnalité peut aussi être utilisée dans un block `switch` :

```
object v = GetValue();
switch (v)
{
    case string s:
        Console.WriteLine($"{v} is a string of length {s.Length}");
        break;
    case int i:
        Console.WriteLine($"{v} is an {(i % 2 == 0 ? "even" : "odd")} int");
        break;
    default:
        Console.WriteLine($"{v} is something else");
        break;
}
```

Dans ce code, chaque `case` introduit une variable du type approprié, qu’on peut ensuite utiliser dans le corps du `case`.

Pour l’instant, je n’ai couvert que le pattern matching avec un simple test du type, mais il existe aussi des formes plus sophistiquées. Par exemple :

```
switch (DateTime.Today)
{
    case DateTime(*, 10, 31):
        Console.WriteLine("Happy Halloween!");
        break;
    case DateTime(var year, 7, 4) when year > 1776:
        Console.WriteLine("Happy Independance Day!");
        break;
    case DateTime { DayOfWeek is DayOfWeek.Saturday }:
    case DateTime { DayOfWeek is DayOfWeek.Sunday }:
        Console.WriteLine("Have a nice week-end!");
        break;
    default:
        break;
}
```

Plutôt cool, non ?

Le pattern matching existe aussi sous une autre forme (encore expérimentale), avec un nouveau mot-clé `match` :

```
object o = GetValue();
string description = o match
    (
        case string s : $"{o} is a string of length {s.Length}"
        case int i : $"{o} is an {(i % 2 == 0 ? "even" : "odd")} int"
        case * : $"{o} is something else"
    );
```

Cela ressemble beaucoup à un `switch`, sauf que c’est une expression et non une instruction.

Il y a encore beaucoup à dire sur le pattern matching, mais ça devient un peu long pour un article de blog, je vous invite donc à consulter [la spec sur Github](https://github.com/dotnet/roslyn/blob/master/docs/features/patterns.md) pour plus de détails.

### Littéraux binaires et séparateurs de chiffres

Ces fonctionnalités n’étaient pas explicitement mentionnées dans les notes de version, mais j’ai remarqué qu’elles étaient inclues aussi. Elles étaient initialement prévues pour C# 6, mais avaient finalement été retirées avant la release ; elles sont de retour en C# 7.

Il est maintenant possible d’écrire des littéraux numériques sous forme binaire, en plus du décimal et de l’hexadécimal :

```
int x = 0b11001010;
```

Pratique pour définir des masques de bits !

De plus, pour rendre les grands nombres plus lisibles, on peut désormais grouper les chiffres en introduisant des séparateurs. Cela fonctionne avec les littéraux décimaux, hexadécimaux, et binaires :

```
int oneBillion = 1_000_000_000;
int foo = 0x7FFF_1234;
int bar = 0b1001_0110_1010_0101;
```

### Conclusion

Visual Studio “15” Preview vous permet donc de commencer à expérimenter avec les nouvelles fonctionnalités de C# 7 ; n’hésitez pas à faire part de vos retours sur Github ! Et gardez à l’esprit que ce n’est qu’une version préliminaire, beaucoup de choses peuvent encore changer avant la version finale.

