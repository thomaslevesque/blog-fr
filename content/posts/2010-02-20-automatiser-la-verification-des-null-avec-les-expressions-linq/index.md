---
layout: post
title: Automatiser la vérification des null avec les expressions Linq
date: 2010-02-20T00:00:00.0000000
url: /2010/02/20/automatiser-la-verification-des-null-avec-les-expressions-linq/
tags:
  - C#
  - expression
  - linq
  - null check
  - proof of concept
categories:
  - Code sample
---

**Le problème**  Je suis sûr qu'il vous est déjà arrivé d'écrire ce genre de code :  
```csharp
X x = GetX();
string name = "Default";
if (xx != null && xx.Foo != null && xx.Foo.Bar != null && xx.Foo.Bar.Baz != null)
{
    name = xx.Foo.Bar.Baz.Name;
}
```
  On veut juste obtenir `name = xx.Foo.Bar.Baz.Name`, mais on est obligé de tester chaque objet intermédiaire pour vérifier qu'il n'est pas nul, ce qui peut vite s'avérer pénible si la propriété voulue est profondément enfouie dans un graphe d'objets...  **Une solution**  Linq offre une fonctionnalité qui permet (entre autres) de régler ce problème : les expressions. Il est possible, à partir d'une expression lambda, d'obtenir son arbre syntaxique (ou AST : Abstract Syntax Tree), et de faire toutes sortes de manipulations sur cet arbre. On peut également générer dynamiquement un arbre syntaxique, et le compiler pour obtenir un delegate qu'on pourra ensuite exécuter.  Mais quel rapport avec le problème qui nous intéresse ? Eh bien tout simplement, nous allons pouvoir utiliser les expressions Linq pour analyser l'arbre syntaxique correspondant à l'accès à la propriété `xx.Foo.Bar.Baz.Name`, et réécrire cet arbre de façon à y ajouter des tests de nullité pour chaque objet intermédiaire.  Nous allons donc créer une méthode d'extension `NullSafeEval`, qui prendra en premier paramètre une expression lambda définissant comment accéder à la propriété voulue, et en deuxième paramètre la valeur par défaut à renvoyer si un objet nul est rencontré en cours de route.  Cette méthode va transformer l'expression `xx.Foo.Bar.Baz.Name` en ceci :  
```csharp
    (xx == null)
    ? defaultValue
    : (xx.Foo == null)
      ? defaultValue
      : (xx.Foo.Bar == null)
        ? defaultValue
        : (xx.Foo.Bar.Baz == null)
          ? defaultValue
          : xx.Foo.Bar.Baz.Name;
```
  Voici l'implémentation de la méthode `NullSafeEval` :  
```csharp
        public static TResult NullSafeEval<TSource, TResult>(this TSource source, Expression<Func<TSource, TResult>> expression, TResult defaultValue)
        {
            var safeExp = Expression.Lambda<Func<TSource, TResult>>(
                NullSafeEvalWrapper(expression.Body, Expression.Constant(defaultValue)),
                expression.Parameters[0]);

            var safeDelegate = safeExp.Compile();
            return safeDelegate(source);
        }

        private static Expression NullSafeEvalWrapper(Expression expr, Expression defaultValue)
        {
            Expression obj;
            Expression safe = expr;

            while (!IsNullSafe(expr, out obj))
            {
                var isNull = Expression.Equal(obj, Expression.Constant(null));

                safe =
                    Expression.Condition
                    (
                        isNull,
                        defaultValue,
                        safe
                    );

                expr = obj;
            }
            return safe;
        }

        private static bool IsNullSafe(Expression expr, out Expression nullableObject)
        {
            nullableObject = null;

            if (expr is MemberExpression || expr is MethodCallExpression)
            {
                Expression obj;
                MemberExpression memberExpr = expr as MemberExpression;
                MethodCallExpression callExpr = expr as MethodCallExpression;

                if (memberExpr != null)
                {
                    // Static fields don't require an instance
                    FieldInfo field = memberExpr.Member as FieldInfo;
                    if (field != null && field.IsStatic)
                        return true;

                    // Static properties don't require an instance
                    PropertyInfo property = memberExpr.Member as PropertyInfo;
                    if (property != null)
                    {
                        MethodInfo getter = property.GetGetMethod();
                        if (getter != null && getter.IsStatic)
                            return true;
                    }
                    obj = memberExpr.Expression;
                }
                else
                {
                    // Static methods don't require an instance
                    if (callExpr.Method.IsStatic)
                        return true;

                    obj = callExpr.Object;
                }

                // Value types can't be null
                if (obj.Type.IsValueType)
                    return true;

                // Instance member access or instance method call is not safe
                nullableObject = obj;
                return false;
            }
            return true;
        }
```
  En résumé, ce code remonte l'arbre de l'expression lambda, en encadrant chaque appel à une propriété ou méthode d'instance par une expression conditionnelle *(condition ? valeur si vrai : valeur si faux)*.  Et voilà comment on utilise cette méthode :  
```csharp
string name = xx.NullSafeEval(x => x.Foo.Bar.Baz.Name, "Default");
```
  C'est tout de même plus clair et plus concis que notre code initial :)  Notez que l'implémentation proposée gère non seulement les accès aux propriétés, mais également les appels de méthode, on pourrait donc avoir quelque chose comme ça :  
```csharp
string name = xx.NullSafeEval(x => x.Foo.GetBar(42).Baz.Name, "Default");
```
  Les indexeurs ne sont pas encore gérés, mais pourraient être ajoutés sans grande difficulté ; je vous laisse le soin de le faire si vous en avez l'usage ;)  **Limitations**  Même si cette solution peut sembler très intéressante au premier abord, lisez la suite avant de vous précipiter pour intégrer ce code dans des programmes réels...  
- Tout d'abord, le code proposé est avant tout un "proof of concept" et n'a pas subi de tests approfondis, sa fiabilité peut donc laisser à désirer.
- Ensuite, il ne faut pas perdre de vue que la génération dynamique de code à partir d'une expression est très pénalisante pour les performances...<br><br>Une piste possible pour limiter ce problème serait de mettre en cache les delegates obtenus pour chaque expression, de façon à ne pas les regénérer inutilement. Malheureusement, il n'y a pas (à ma connaissance) de moyen simple de comparer deux expressions Linq, ce qui complique sensiblement l'implémentation de ce cache...
- D'autre part, vous aurez peut-être remarqué que les propriétés et méthodes intermédiaires de l'expression sont évaluées plusieurs fois ; non seulement cela peut avoir un impact non négligeable sur les performances, mais surtout, cela peut causer des effets de bord aux conséquences difficilement prévisibles...<br><br>Une solution possible serait de réécrire l'expression de la façon suivante :<br><br>
```csharp
Foo foo = null;
Bar bar = null;
Baz baz = null;
var name =
    (x == null)
    ? defaultValue
    : ((foo = x.Foo) == null)
      ? defaultValue
      : ((bar = foo.Bar) == null)
        ? defaultValue
        : ((baz = bar.Baz) == null)
          ? defaultValue
          : baz.Name;
```
<br><br>Malheureusement, ce n'est pas possible en .NET 3.5 : en effet, cette version ne supporte que des expressions simples, il n'est donc pas possible de déclarer des variables ou de leur affecter une valeur, ni d'écrire plusieurs instructions distinctes. En revanche, en .NET 4.0, le support des expressions Linq a été grandement amélioré, et il est possible de générer ce type de code. J'ai commencé à transformer le code de `NullSafeEval` pour arriver à ce résultat, mais ça s'avère beaucoup plus complexe que prévu... je publierai la nouvelle méthode si j'arrive à quelque chose de concluant ;)

Au final, je ne recommande donc pas d'utiliser cette technique dans du code réel, en tous cas en l'état actuel. Cela donne cependant un aperçu intéressant des possibilités des expressions Linq. Sachez qu'elle sont également utilisées, entre autres :
- Pour la génération de requêtes SQL dans des ORM comme Linq to SQL ou Entity Framework
- Pour construire dynamiquement des prédicats complexes, comme dans la classe [PredicateBuilder](http://www.albahari.com/nutshell/predicatebuilder.aspx) de Joseph Albahari
- Pour implementer la "réflexion statique" dont on a [beaucoup parlé sur les blogs](http://www.google.com/search?tbo=1&amp;tbs=blg:1&amp;q=static+reflection) depuis quelques temps


