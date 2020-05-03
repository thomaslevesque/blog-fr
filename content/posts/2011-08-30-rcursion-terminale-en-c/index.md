---
layout: post
title: Récursion terminale en C#
date: 2011-08-30T00:00:00.0000000
url: /2011/08/30/rcursion-terminale-en-c/
tags:
  - C#
  - récursion terminale
  - tail recursion
  - trampoline
categories:
  - Code sample
---


Quel que soit le langage de programmation utilisé, certains traitements s’implémentent naturellement sous forme d’un algorithme récursif (même si ce n’est pas toujours la solution la plus optimale). Le problème de l’approche récursive, c’est qu’elle consomme potentiellement beaucoup d’espace sur la pile : à partir d’un certain niveau de “profondeur” de la récursion, l’espace alloué pour la pile d’exécution du thread est épuisé, et on obtient une erreur de type “débordement de la pile” (`StackOverflowException` en .NET).
<!--more-->
###### **Récursion terminale ? Mais qu’est-ce que c’est ?**

Certains langages, notamment les langages fonctionnels, proposent nativement une optimisation appelée “[récursion terminale](http://fr.wikipedia.org/wiki/R%C3%A9cursion_terminale)” (*tail recursion* en anglais). Le principe est que si l’appel récursif est la dernière instruction d’une fonction, il n’est pas nécessaire de conserver sur la pile le contexte de l’appel courant, puisqu’on aura pas à y revenir : il suffit de remplacer les paramètres par leurs nouvelles valeurs, et de reprendre l’exécution au début de la fonction. La récursion est donc transformée en itération, si bien qu’on ne risque plus de provoquer un débordement de la pile. Le concept étant assez nouveau pour moi, je ne vais pas faire un cours complet sur la récursion terminale… des personnes beaucoup plus compétentes s’en sont déjà chargées ! Je vous suggère de suivre le lien Wikipedia ci-dessus, qui est un bon point de départ pour comprendre cette notion.

En C#, le compilateur n’implémente malheureusement pas la récursion terminale, ce qui est un peu dommage dans la mesure où [le CLR le supporte](http://msdn.microsoft.com/en-us/library/system.reflection.emit.opcodes.tailcall.aspx)… Pourtant, tout n’est pas perdu ! Certaines personnes ont eu une idée très astucieuse pour résoudre le problème : une technique appelée “trampoline” (parce qu’elle fait “rebondir” la fonction) qui permet de transformer très facilement un algorithme récursif en algorithme itératif. Samuel Jack explique très bien le concept [sur son blog](http://blog.functionalfun.net/2008/04/bouncing-on-your-tail.html) (en anglais). Dans la suite de cet article, on verra comment appliquer cette technique à un algorithme simple, en utilisant la classe présentée dans l’article de Samuel Jack ; et pour finir, je présenterai une autre implémentation d’un trampoline, qui me semble un peu plus souple.

###### **Un cas d’utilisation en C#**

Voyons donc comment on peut transformer un algorithme récursif simple, comme le calcul de la factorielle d’un nombre, en un algorithme qui utilise la récursion terminale (soit dit en passant, on peut calculer la factorielle beaucoup plus efficacement avec un algorithme non récursif, mais pour l’exemple on fera comme si on ne le savait pas…). L’algorithme qui découle directement de la définition est le suivant :

```
BigInteger Factorial(int n)
{
    if (n < 2)
        return 1;
    return n * Factorial(n - 1);
}
```

(Notez l’utilisation de `BigInteger` : si on veut pousser la récursion assez loin pour observer les effets de la récursion terminale, le résultat dépassera largement la capacité d’un `long`)

Si on appelle cette méthode avec une grande valeur (aux alentours de 20000 sur ma machine), on obtient l’exception à laquelle on pouvait s’attendre : `StackOverflowException`. On a “empilé” tellement d’appels imbriqués à la méthode `Factorial` qu’on a épuisé la capacité de la pile. On va donc essayer de modifier le code pour pouvoir utiliser la récursion terminale.

Comme mentionné plus haut, il faut que la dernière instruction de la méthode soit l’appel récursif à elle-même. Cela *semble* être le cas ici… mais en fait non : la dernière opération exécutée sera la multiplication, qui ne peut pas être effectuée avant de connaitre le résultat de `Factorial(n-1)`. Il faut donc repenser un peu la méthode, pour qu’elle se termine par un appel à elle-même avec des paramètres différents. Pour arriver à ce résultat, on va ajouter un paramètre `product`, qui va jouer le rôle d’accumulateur :

```
BigInteger Factorial(int n, BigInteger product)
{
    if (n < 2)
        return product;
    return Factorial(n - 1, n * product);
}
```

Pour le premier appel, il suffira de passer 1 comme valeur initiale de l’accumulateur.

On a donc maintenant une méthode qui remplit les critères pour la récursion terminale : l’appel récursif à `Factorial` est bien la dernière opération effectuée par la méthode. Une fois qu’on a mis l’algorithme sous cette forme, la transformation finale pour appliquer la récursion terminale à l’aide du trampoline de Samuel Jack est triviale :

```
Bounce<int, BigInteger, BigInteger> Factorial(int n, BigInteger product)
{
    if (n < 2)
        return Trampoline.ReturnResult<int, BigInteger, BigInteger>(product);
    return Trampoline.Recurse<int, BigInteger, BigInteger>(n - 1, n * product);
}
```

- Le renvoi de la valeur finale est remplacé par un appel à `Trampoline.ReturnResult`, qui indique au trampoline que le calcul est terminé avec le résultat indiqué
- L’appel récursif à `Factorial` est remplacé par un appel à `Trampoline.Recurse`, qui indique les paramètres à utiliser pour le prochain appel de la méthode


Cette méthode n’est pas directement utilisable, puisqu’elle renvoie un objet `Bounce` dont on ne sait pas trop quoi faire… Pour l’exécuter, on utilise la méthode `Trampoline.MakeTrampoline`, qui renvoie une nouvelle fonction qui applique la récursion terminale. On peut ensuite exécuter cette fonction directement :

```
    Func<int, BigInteger, BigInteger> fact = Trampoline.MakeTrampoline<int, BigInteger, BigInteger>(Factorial);
    BigInteger result = fact(50000, 1);
```

On peut maintenant calculer la factorielle d’un grand nombre sans risque de “faire sauter la pile”… bien sûr, ça reste assez lent : comme mentionné plus haut, ce n’est pas du tout la façon optimale de calculer une factorielle, et de plus les opérations avec `BigInteger` sont assez lourdes.

###### **Peut-on faire mieux ?**

Vous vous doutez bien que si je pose la question, c’est que la réponse est oui… L’implémentation du trampoline présentée ci-dessus remplit parfaitement son rôle, mais personnellement je trouve qu’elle manque de souplesse :

- Elle ne fonctionne qu’avec 2 paramètres (on peut bien sûr l’adapter pour un nombre différent de paramètres, mais il faut à chaque fois créer de nouvelles méthodes avec les signatures adéquates)
- L’écriture est assez lourde : il y a 3 arguments de type, qu’il faut préciser à chaque fois parce que l’inférence de type n’a pas tous les éléments nécessaires pour les déterminer automatiquement
- Le fait que `MakeTrampoline` renvoie une nouvelle fonction ne me semble pas très utile, il serait plus intuitif d’avoir une méthode `Execute` qui exécute directement la fonction


Et pour finir, je trouve que la terminologie, bien qu’amusante, n’est pas très explicite…

J’ai donc cherché à améliorer ce système pour le rendre plus pratique, en utilisant des méthodes anonymes sous forme d’expressions lambda. Il n’y a plus qu’un seul argument de type (le type de retour), et on passe les paramètres par closure dans une expression lambda. Voilà ce que donne la nouvelle version de la méthode `Factorial` avec mon implémentation :

```
RecursionResult<BigInteger> Factorial(int n, BigInteger product)
{
    if (n < 2)
        return TailRecursion.Return(product);
    return TailRecursion.Next(() => Factorial(n - 1, n * product));
}
```

Et on l’utilise comme ceci :

```
BigInteger result = TailRecursion.Execute(() => Factorial(50000, 1));
```

C’est plus souple, plus concis, et plus lisible… enfin il me semble en tous cas![Sourire](wlemoticon-smile.png). Le revers de la médaille, c’est que performances sont légèrement moins bonnes (environ 20% plus long pour calculer la factorielle de 50000), probablement à cause du delegate qui est créé à chaque niveau de récursion.

Voici le code complet de la classe `TailRecursion` :

```
public static class TailRecursion
{
    public static T Execute<T>(Func<RecursionResult<T>> func)
    {
        do
        {
            var recursionResult = func();
            if (recursionResult.IsFinalResult)
                return recursionResult.Result;
            func = recursionResult.NextStep;
        } while (true);
    }
    
    public static RecursionResult<T> Return<T>(T result)
    {
        return new RecursionResult<T>(true, result, null);
    }
    
    public static RecursionResult<T> Next<T>(Func<RecursionResult<T>> nextStep)
    {
        return new RecursionResult<T>(false, default(T), nextStep);
    }

}

public class RecursionResult<T>
{
    private readonly bool _isFinalResult;
    private readonly T _result;
    private readonly Func<RecursionResult<T>> _nextStep;
    internal RecursionResult(bool isFinalResult, T result, Func<RecursionResult<T>> nextStep)
    {
        _isFinalResult = isFinalResult;
        _result = result;
        _nextStep = nextStep;
    }
    
    public bool IsFinalResult { get { return _isFinalResult; } }
    public T Result { get { return _result; } }
    public Func<RecursionResult<T>> NextStep { get { return _nextStep; } }
}
```

###### **Peut-on aller plus loin ?**

Certainement… mais ça devient de la haute voltige ! Comme je le disais plus haut, le CLR supporte la récursion terminale, via l’instruction IL `tail`. L’idéal serait que le compilateur C# sache générer cette instruction quand une méthode est éligible à la récursion terminale, mais ce n’est malheureusement pas le cas, et je ne pense pas qu’il faille s’attendre à ce que ce soit géré dans une prochaine version, car la demande pour cette fonctionnalité semble assez faible.

En attendant, on peut tricher un peu, en aidant le compilateur à faire son travail : le .NET framework fournit des outils nommés [ildasm](http://msdn.microsoft.com/en-us/library/f7dy01k1.aspx) (désassembleur IL) et [ilasm](http://msdn.microsoft.com/en-us/library/496e4ekx.aspx) (assembleur IL), avec lesquels on peut s’amuser un peu… Reprenons notre implémentation récursive classique qui n’utilise pas encore la récursion terminale :

```
	static BigInteger Factorial(int n, BigInteger product){	if (n < 2)		return product;	return Factorial(n - 1, n * product);}
```

Si on compile ce code, et qu’on le désassemble avec idlasm, on obtient le code IL suivant :

```
  .method private hidebysig static valuetype [System.Numerics]System.Numerics.BigInteger 
          Factorial(int32 n,
                    valuetype [System.Numerics]System.Numerics.BigInteger product) cil managed
  {
    // Code size       41 (0x29)
    .maxstack  3
    .locals init (valuetype [System.Numerics]System.Numerics.BigInteger V_0,
             bool V_1)
    IL_0000:  nop
    IL_0001:  ldarg.0
    IL_0002:  ldc.i4.2
    IL_0003:  clt
    IL_0005:  ldc.i4.0
    IL_0006:  ceq
    IL_0008:  stloc.1
    IL_0009:  ldloc.1
    IL_000a:  brtrue.s   IL_0010

    IL_000c:  ldarg.1
    IL_000d:  stloc.0
    IL_000e:  br.s       IL_0027

    IL_0010:  ldarg.0
    IL_0011:  ldc.i4.1
    IL_0012:  sub
    IL_0013:  ldarg.0
    IL_0014:  call       valuetype [System.Numerics]System.Numerics.BigInteger [System.Numerics]System.Numerics.BigInteger::op_Implicit(int32)
    IL_0019:  ldarg.1
    IL_001a:  call       valuetype [System.Numerics]System.Numerics.BigInteger [System.Numerics]System.Numerics.BigInteger::op_Multiply(valuetype [System.Numerics]System.Numerics.BigInteger,
                                                                                                                                        valuetype [System.Numerics]System.Numerics.BigInteger)
    IL_001f:  call       valuetype [System.Numerics]System.Numerics.BigInteger Program::Factorial(int32,
                                                                                                  valuetype [System.Numerics]System.Numerics.BigInteger)
    IL_0024:  stloc.0
    IL_0025:  br.s       IL_0027

    IL_0027:  ldloc.0
    IL_0028:  ret
  } // end of method Program::Factorial
```

Ca pique un peu les yeux si on a pas l’habitude de lire du code IL, mais on arrive à peu près à voir ce qui se passe… L’appel récursif se situe à l’adresse `IL_001f`, c’est à ce niveau là qu’on va pouvoir intervenir. Si on lit la documentation de l’instruction `tail`, on apprend qu’elle doit précéder immédiatement une instruction `call`, et que l’instruction suivant le `call` doit être `ret` (retour de la méthode). Pour l’instant, il y a plusieurs instructions après le `call`, parce que le compilateur a généré une variable locale pour stocker la valeur de retour. Il faut juste modifier légèrement le code pour ne plus utiliser cette variable, et ajouter l’instruction `tail` au bon endroit :

```
  .method private hidebysig static valuetype [System.Numerics]System.Numerics.BigInteger 
          Factorial(int32 n,
                    valuetype [System.Numerics]System.Numerics.BigInteger product) cil managed
  {
    // Code size       41 (0x29)
    .maxstack  3
    .locals init (valuetype [System.Numerics]System.Numerics.BigInteger V_0,
             bool V_1)
    IL_0000:  nop
    IL_0001:  ldarg.0
    IL_0002:  ldc.i4.2
    IL_0003:  clt
    IL_0005:  ldc.i4.0
    IL_0006:  ceq
    IL_0008:  stloc.1
    IL_0009:  ldloc.1
    IL_000a:  brtrue.s   IL_0010

    IL_000c:  ldarg.1
    IL_000d:  ret		// Return directly instead of storing the result in V_0
    IL_000e:  nop

    IL_0010:  ldarg.0
    IL_0011:  ldc.i4.1
    IL_0012:  sub
    IL_0013:  ldarg.0
    IL_0014:  call       valuetype [System.Numerics]System.Numerics.BigInteger [System.Numerics]System.Numerics.BigInteger::op_Implicit(int32)
    IL_0019:  ldarg.1
    IL_001a:  call       valuetype [System.Numerics]System.Numerics.BigInteger [System.Numerics]System.Numerics.BigInteger::op_Multiply(valuetype [System.Numerics]System.Numerics.BigInteger,
                                                                                                                                        valuetype [System.Numerics]System.Numerics.BigInteger)
    IL_001f:  tail.
    IL_0020:  call       valuetype [System.Numerics]System.Numerics.BigInteger Program::Factorial(int32,
                                                                                                  valuetype [System.Numerics]System.Numerics.BigInteger)
    IL_0025:  ret		// Return directly instead of storing the result in V_0

  } // end of method Program::Factorial
```
 On réassemble ce code avec ilasm, et on obtient un nouvel exécutable, qui s’exécute sans problème même avec des valeurs pour lesquelles la version classique aurait planté depuis longtemps ![Sourire](wlemoticon-smile.png). Les performances sont également très bonnes : environ 3 fois plus rapide qu’avec la classe Trampoline. Si on compare les performances avec des valeurs plus petites, on remarque que c’est également 3 fois plus rapide que la version classique sans récursion terminale.   
Bien sûr, c’est n’est qu’une preuve de concept… il ne semble pas très réaliste d'effectuer cette opération manuellement dans un “vrai” projet. En revanche, il serait probablement possible de créer un outil qui modifie l’assembly automatiquement après la compilation pour ajouter la récursion terminale.

