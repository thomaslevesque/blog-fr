---
layout: post
title: 'Filtres d’exception en C# 6 : leur plus grand avantage n’est pas celui qu’on croit'
date: 2015-06-23T00:00:00.0000000
url: /2015/06/23/filtres-dexception-en-c-6-leur-plus-grand-avantage-nest-pas-celui-quon-croit/
tags:
  - C# 6
  - filtres d'exception
  - pile
categories:
  - Uncategorized
---


Les filtres d’exception sont l’une des fonctionnalités majeures de C# 6. Ils tirent parti d’une fonctionnalité du CLR qui a toujours existé, mais qui n’était pas exploitée en C# jusqu’ici. Ils permettent de spécifier une condition sur un bloc `catch` :

```
static void Main()
{
    try
    {
        Foo.DoSomethingThatMightFail(null);
    }
    catch (MyException ex) when (ex.Code == 42)
    {
        Console.WriteLine("Error 42 occurred");
    }
}
```

Comme on pourrait s’y attendre, le bloc `catch` ne sera exécuté que si `ex.Code == 42`. Si cette condition n’est pas vérifiée, l’exception sera propagée en remontant la pile jusqu’à ce qu’elle soit interceptée ailleurs ou qu’elle termine le processus.

A première vue, cela n’apporte rien de très nouveau. Après tout, on pouvait déjà faire ceci :

```
static void Main()
{
    try
    {
        Foo.DoSomethingThatMightFail(null);
    }
    catch (MyException ex)
    {
        if (ex.Code == 42)
            Console.WriteLine("Error 42 occurred");
        else
            throw;
    }
}
```

Puisque ce bout de code est équivalent au précédent, les filtres d’exception sont juste du sucre syntaxique… enfin, ils *sont* équivalents, non ?

NON !

### Déroulement de pile

Il y a en fait une différence subtile mais importante : **les filtres d’exception ne déroulent pas la pile**. OK, mais qu’est-ce que ça veut dire ?

Quand on entre dans un bloc catch, la pile est déroulée : cela signifie que toutes les frames pour les appels de méthode plus “profonds” que la méthode courante sont dépilées. Cela implique que toutes les informations concernant l’état d’exécution de ces méthodes sont perdues, ce qui rend plus difficile de trouver la cause première de l’exception.

Supposons que la méthode `DoSomethingThatMightFail` lance une `MyException` avec le code 123, et que le débogueur soit configuré pour ne s’arrêter que sur les exceptions non gérées.

- Dans le code sans filtre d’exception, on entre toujours dans le bloc `catch` (puisque le type de l’exception correspond), et la pile est immédiatement déroulée. Puisque l’exception ne satisfait pas la condition, elle est relancée. Le débogueur s’arrêtera donc sur le `throw;` dans le block `catch` ; aucune information sur l’état d’exécution de la méthode `DoSomethingThatMightFail` ne sera disponible. Autrement dit, on ne pourra pas savoir ce qui était en train de se passer dans la méthode qui a lancé l’exception.
- En revanche, dans le code qui utilise un filtre d’exception, l’exception ne satisfait pas la condition, donc on n’entrera pas du tout dans le bloc `catch`, et la pile ne sera pas déroulée. Le débogueur s’arrêtera dans la méthode `DoSomethingThatMightFail`, ce qui permettra de voir facilement ce qui était en train de se passer quand l’exception a été lancée.


Bien sûr, quand on débogue directement une application dans Visual Studio, on peut configurer le débogueur pour s’arrêter dès qu’une exception est lancée, qu’elle soit gérée ou non. Mais on n’a pas toujours cette possibilité ; par exemple, si on débogue un problème en production, on travaille plutôt sur un crash dump, donc le fait que la pile n’ait pas été déroulée devient très utile, puisque ça permet de voir ce qui était en train de se passer dans la méthode qui a lancé l’exception.

### Pile vs. trace de pile

Vous avez peut-être remarqué que j’ai parlé plus haut de la pile (call stack), et non de la trace de pile (stack trace). Bien qu’on utilise souvent le terme “pile” pour faire référence à la trace de pile, ce sont deux choses bien différentes. La pile est une zone mémoire allouée à chaque thread qui contient des informations sur les méthodes en cours d’exécution : adresse de retour, arguments, et variables locales. La trace de pile est juste une chaine qui contient les noms des méthodes actuellement sur la pile (et l’emplacement dans ces méthodes, si les symboles de débogage sont disponibles). La propriété `Exception.StackTrace` contient la trace de la pile telle qu’elle était quand l’exception a été lancée, et n’est pas affectée quand la pile est déroulée ; si on relance l’exception avec `throw;`, elle n’est pas modifiée non plus. Elle n’est écrasée que si on relance l’exception avec `throw ex;`. La pile elle-même, en revanche, est déroulée quand on entre dans un bloc `catch`, comme décrit plus haut.

### Effets de bord

il est intéressant de noter qu’un filtre d’exception peut contenir n’importe quelle expression qui renvoie un `bool` (enfin presque… on ne peut pas utiliser `await` par exemple). Cela peut être une condition logique, une propriété, un appel de méthode, etc. Techniquement, rien n’empêche de causer des effets de bord dans un filtre d’exception. Dans la plupart des cas, je déconseillerais vivement de faire ça, car ça peut causer des comportements très déroutants ; il peut devenir très difficile de comprendre dans quel ordre les choses sont exécutées. Cependant, il y a un scénario courant qui pourrait bénéficier d’effets de bord dans un filtre d’exception: le logging. On pourrait facilement créer une méthode qui log l’exception et renvoie false pour qu’on n’entre pas dans le bloc `catch`. Cela permettrait de logger les exception à la volée sans les gérer, et donc sans dérouler la pile:

```
try
{
    DoSomethingThatMightFail(s);
}
catch (Exception ex) when (Log(ex, "An error occurred"))
{
    // this catch block will never be reached
}
 
...
 
static bool Log(Exception ex, string message, params object[] args)
{
    Debug.Print(message, args);
    return false;
}
```

### Conclusion

Comme avez pu le voir, les filtres d’exception ne sont pas juste du sucre syntaxique. Contrairement à la plupart des fonctionnalités de C# 6, ce n’est pas vraiment une fonctionnalité de “codage” (dans le sens où ça ne rend pas le code significativement plus clair), mais plutôt une fonctionnalité de “débogage”. Bien compris et utilisés, ils peuvent rendre beaucoup plus facile la résolution de problèmes dans le code.

