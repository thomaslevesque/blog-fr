---
layout: post
title: Un moyen facile de tester unitairement la validation des arguments null
date: 2014-11-02T00:00:00.0000000
url: /2014/11/02/easy-unit-testing-of-null-argument-validation/
tags:
  - C#
  - nunit
  - tests unitaires
  - validation d'argument
categories:
  - Uncategorized
---


Quand on teste unitairement une méthode, une des choses à tester est la validation des arguments : par exemple, vérifier que la méthode lève bien une `ArgumentNullException` quand un argument null est passé pour un paramètre qui ne doit pas être null. Ecrire ce genre de test est très facile, mais c’est une tâche fastidieuse et répétitive, surtout pour une méthode qui a beaucoup de paramètres. J’ai donc écrit une méthode qui automatise en partie cette tâche : elle essaie de passer null pour chacun des arguments spécifiés, et vérifie que la méthode lève bien une `ArgumentNullException`. Voici un exemple qui teste une méthode d’extension `FullOuterJoin` :

```
[Test]
public void FullOuterJoin_Throws_If_Argument_Null()
{
    var left = Enumerable.Empty<int>();
    var right = Enumerable.Empty<int>();
    TestHelper.AssertThrowsWhenArgumentNull(
        () => left.FullOuterJoin(right, x => x, y => y, (k, x, y) => 0, 0, 0, null),
        "left", "right", "leftKeySelector", "rightKeySelector", "resultSelector");
}
```

Le premier paramètre est une expression lambda qui représente la façon d’appeler la méthode testée. Dans cette lambda, tous les arguments passés à la méthode doivent être valides. Les paramètres suivants sont les noms des paramètres qui ne doivent pas être null. Pour chacun des noms spécifiés, `AssertThrowsWhenArgumentNull` va remplacer l’argument correspondant par null dans l’expression lambda, compiler et invoquer l’expression lambda, et vérifier que la méthode lève bien une `ArgumentNullException`.

Grâce à cette méthode, au lieu d’écrire un test pour chacun des arguments à valider, il suffit d’un seul test.

Voici le code de la méthode `TestHelper.AssertThrowsWhenArgumentNull` (vous pouvez aussi le trouver sur [Gist](https://gist.github.com/thomaslevesque/c4cb9f537316b122f5b9)):

```
using System;
using System.Linq;
using System.Linq.Expressions;
using NUnit.Framework;

namespace MyLibrary.Tests
{
    static class TestHelper
    {
        public static void AssertThrowsWhenArgumentNull(Expression<TestDelegate> expr, params string[] paramNames)
        {
            var realCall = expr.Body as MethodCallExpression;
            if (realCall == null)
                throw new ArgumentException("Expression body is not a method call", "expr");

            var realArgs = realCall.Arguments;
            var paramIndexes = realCall.Method.GetParameters()
                .Select((p, i) => new { p, i })
                .ToDictionary(x => x.p.Name, x => x.i);
            var paramTypes = realCall.Method.GetParameters()
                .ToDictionary(p => p.Name, p => p.ParameterType);
            
            

            foreach (var paramName in paramNames)
            {
                var args = realArgs.ToArray();
                args[paramIndexes[paramName]] = Expression.Constant(null, paramTypes[paramName]);
                var call = Expression.Call(realCall.Method, args);
                var lambda = Expression.Lambda<TestDelegate>(call);
                var action = lambda.Compile();
                var ex = Assert.Throws<ArgumentNullException>(action, "Expected ArgumentNullException for parameter '{0}', but none was thrown.", paramName);
                Assert.AreEqual(paramName, ex.ParamName);
            }
        }

    }
}
```

Notez que cette méthode a été écrite pour NUnit, mais vous pouvez facilement l’adapter à d’autres frameworks de test unitaire.

J’ai utilisé cette méthode dans ma librairie [Linq.Extras](https://github.com/thomaslevesque/Linq.Extras), qui fournit de nombreuses méthodes d’extension supplémentaires pour travailler avec des séquences et collections (elle inclut par exemple la méthode `FullOuterJoin` mentionnée plus haut).

