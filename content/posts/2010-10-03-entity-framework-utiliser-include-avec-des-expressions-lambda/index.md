---
layout: post
title: '[Entity Framework] Utiliser Include avec des expressions lambda'
date: 2010-10-03T00:00:00.0000000
url: /2010/10/03/entity-framework-utiliser-include-avec-des-expressions-lambda/
tags:
  - .NET 4.0
  - Entity Framework
  - expression
  - include
  - lambda
  - linq
categories:
  - C# 4.0
  - Code sample
---

Je travaille en ce moment sur un projet qui utilise Entity Framework 4. Bien que le lazy loading soit activé, j'utilise généralement la méthode [`ObjectQuery.Include`](http://msdn.microsoft.com/en-us/library/bb738708.aspx) pour charger les entités associées en une seule fois, de façon à éviter des appels supplémentaires à la base de données lors de l'accès à ces entités :  
```csharp
var query =
    from ord in db.Orders.Include("OrderDetails")
    where ord.Date >= DateTime.Today
    select ord;
```
  Ou encore, pour inclure aussi le produit :  
```csharp
var query =
    from ord in db.Orders.Include("OrderDetails.Product")
    where ord.Date >= DateTime.Today
    select ord;
```
  Il y a quelque chose qui m'ennuie avec cette méthode `Include` : le fait de devoir spécifier le chemin de la propriété sous forme de chaine de caractères. En effet cette approche présente deux inconvénients majeurs : 
- Elle comporte un risque d'erreur important : on a vite fait de faire une faute de frappe dans le chemin de la propriété, et puisque c'est une chaine de caractères, le compilateur ne remarque rien. On a donc une erreur à l'exécution alors que ça aurait pu être vérifié dès la compilation.
- On ne profite plus de l'assistance de l'IDE : pas d'intellisense ni de refactoring. Si on renomme une propriété du modèle, le refactoring automatique ne prend pas en compte le contenu des chaines de caractères. Il faut donc aller modifier manuellement les appels à `Include` qui font référence à cette propriété, avec le risque non négligeable d'en oublier au passage...

  Il serait donc plus pratique d'utiliser une expression lambda pour spécifier le chemin de la propriété à inclure. Le principe est connu, et fréquemment utilisé pour éviter de passer une chaine quand on veut spécifier le nom d'une propriété.  Le cas "de base", dans lequel on ne charge qu'une propriété directement liée à la source, est assez simple à gérer, et on trouve des implémentations un peu partout sur le net. Il suffit d'utiliser une méthode qui extrait le nom de la propriété à partir de l'expression :  
```csharp
    public static class ObjectQueryExtensions
    {
        public static ObjectQuery<T> Include<T>(this ObjectQuery<T> query, Expression<Func<T, object>> selector)
        {
            string propertyName = GetPropertyName(selector);
            return query.Include(propertyName);
        }

        private static string GetPropertyName<T>(Expression<Func<T, object>> expression)
        {
            MemberExpression memberExpr = expression.Body as MemberExpression;
            if (memberExpr == null)
                throw new ArgumentException("Expression body must be a member expression");
            return memberExpr.Member.Name;
        }
    }
```
  En utilisant cette méthode d'extension, on peut réécrire le code du premier exemple de la façon suivante :  
```csharp
var query =
    from ord in db.Orders.Include(o => o.OrderDetails)
    where ord.Date >= DateTime.Today
    select ord;
```
  Ce code fonctionne, mais seulement pour les cas simples... dans le deuxième exemple, on veut aussi inclure la propriété `OrderDetail.Product`, et le code ci-dessus ne permet pas de gérer ce cas. En effet, l'expression qu'il faudrait écrire pour inclure la propriété `Product` serait du type `o.OrderDetails.Select(od => od.Product)`, or la méthode `GetPropertyName` ne sait gérer que les propriétés, pas les appels de méthode...  Pour obtenir le chemin complet de la propriété à inclure, il faut parcourir tout l'arbre d'expression pour en extraire les propriétés. Bien que cela puisse paraitre assez complexe, il existe une classe qui peut nous y aider : [`ExpressionVisitor`](http://msdn.microsoft.com/en-us/library/system.linq.expressions.expressionvisitor.aspx). Cette classe, introduite en .NET 4.0, implémente le design pattern Visiteur pour parcourir tous les noeuds de l'arbre. L'implémentation de base ne fait rien de particulier, elle se contente de visiter chaque noeud. Tout ce que nous avons à faire, c'est en hériter pour spécialiser certaines méthodes de façon à extraire les propriétés utilisées dans l'expression. On va donc redéfinir les méthodes suivantes :
- `VisitMember` : c'est la méthode appelée pour visiter l'accès à une propriété ou à un champ
- `VisitMethodCall` : la méthode appelée pour visiter les appels de méthode. Bien que ce cas ne nous intéresse pas directement a priori, on doit modifier son comportement dans le cas des opérateurs Linq : l'implémentation par défaut visite les paramètres dans l'ordre normal, mais pour les méthodes d'extension comme `Select` ou `SelectMany`, on doit visiter le premier paramètre (le paramètre `this`) en dernier, de façon à conserver l'ordre voulu pour le chemin de la propriété

  Voici donc la nouvelle implémentation de la méthode d'extension `Include` :  
```csharp
    public static class ObjectQueryExtensions
    {
        public static ObjectQuery<T> Include<T>(this ObjectQuery<T> query, Expression<Func<T, object>> selector)
        {
            string path = new PropertyPathVisitor().GetPropertyPath(expression);
            return query.Include(path);
        }

        class PropertyPathVisitor : ExpressionVisitor
        {
            private Stack<string> _stack;

            public string GetPropertyPath(Expression expression)
            {
                _stack = new Stack<string>();
                Visit(expression);
                return _stack
                    .Aggregate(
                        new StringBuilder(),
                        (sb, name) =>
                            (sb.Length > 0 ? sb.Append(".") : sb).Append(name))
                    .ToString();
            }

            protected override Expression VisitMember(MemberExpression expression)
            {
                if (_stack != null)
                    _stack.Push(expression.Member.Name);
                return base.VisitMember(expression);
            }

            protected override Expression VisitMethodCall(MethodCallExpression expression)
            {
                if (IsLinqOperator(expression.Method))
                {
                    for (int i = 1; i < expression.Arguments.Count; i++)
                    {
                        Visit(expression.Arguments[i]);
                    }
                    Visit(expression.Arguments[0]);
                    return expression;
                }
                return base.VisitMethodCall(expression);
            }

            private static bool IsLinqOperator(MethodInfo method)
            {
                if (method.DeclaringType != typeof(Queryable) && method.DeclaringType != typeof(Enumerable))
                    return false;
                return Attribute.GetCustomAttribute(method, typeof(ExtensionAttribute)) != null;
            }
        }
    }
```
  J'ai déjà parlé plus haut de la méthode `VisitMethodCall`, je ne reviens donc pas dessus. L'implémentation de `VisitMember` est très simple : on se contente d'ajouter le nom de la propriété sur une pile. Au fait, pourquoi une pile ? Parce que la visite de l'expression ne se déroule pas dans l'ordre auquel on pense intuitivement. Par exemple, dans une expression du type `o.OrderDetails.Select(od => od.Product)`, le premier noeud examiné n'est pas `o`, mais l'appel à `Select`, car ce qui précède (`o.OrderDetails`) est en fait un paramètre de la méthode statique `Select`... Pour obtenir les propriétés dans l'ordre voulu, on les place donc sur une pile de façon à les relire ensuite dans l'ordre inverse.  La méthode `GetPropertyPath` est elle aussi assez facile à comprendre : elle initialise la pile, visite l'expression, et reconstitue le chemin à partir de la pile.  On peut donc maintenant réécrire le code du deuxième exemple de la façon suivante :  
```csharp
var query =
    from ord in db.Orders.Include(o => OrderDetails.Select(od => od.Product))
    where ord.Date >= DateTime.Today
    select ord;
```
  Cette méthode fonctionne aussi pour des cas plus complexes. Ajoutons un peu de piment à notre exemple : une ou plusieurs remises peuvent être appliquées à chaque article commandé, et chaque remise est associée à une campagne promotionnelle. Si on veut inclure les remises et les campagnes associées dans les résultats de la requête, on peut écrire quelque chose comme ça :  
```csharp
var query =
    from ord in db.Orders.Include(o => OrderDetails.Select(od => od.Discounts.Select(d => d.Campaign)))
    where ord.Date >= DateTime.Today
    select ord;
```
  Le résultat est le même que si on avait passé à `Include` le chemin "OrderDetails.Discounts.Campaign".  Comme les `Select` imbriqués réduisent la lisibilité du code, on peut écrire l'expression un peu différemment :  
```csharp
var query =
    from ord in db.Orders.Include(o => o.OrderDetails
                                        .SelectMany(od => od.Discounts)
                                        .Select(d => d.Campaign))
    where ord.Date >= DateTime.Today
    select ord;
```
Pour finir, deux remarques sur cette solution :
- Une méthode d'extension similaire est inclue dans le [Entity Framework Feature CTP4](http://blogs.msdn.com/b/adonet/archive/2010/07/14/ctp4announcement.aspx) (voir [cet article](http://romiller.com/2010/07/14/ef-ctp4-tips-tricks-include-with-lambda/) pour plus de détails). Il est donc probable qu'elle finisse par être intégrée dans le Framework (peut-être dans un service pack pour .NET 4.0 ?)
- Bien que cette solution cible Entity Framework 4.0, il est a priori possible de l'adapter à EF 3.5. La classe `ExpressionVisitor` n'est pas disponible en 3.5, mais le [LINQKit](http://www.albahari.com/nutshell/linqkit.aspx) de Joseph Albahari en fournit une implémentation. Je n'ai pas essayé, mais ça devrait fonctionner de la même façon...


