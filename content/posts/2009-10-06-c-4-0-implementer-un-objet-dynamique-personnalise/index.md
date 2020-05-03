---
layout: post
title: '[C# 4.0] Implémenter un objet dynamique personnalisé'
date: 2009-10-06T00:00:00.0000000
url: /2009/10/06/c-4-0-implementer-un-objet-dynamique-personnalise/
tags:
  - .NET 4.0
  - C# 4.0
  - DLR
  - dynamic
categories:
  - C# 4.0
  - Code sample
---


Comme vous le savez sans doute déjà si vous vous intéressez à l’actualité de .NET, la future version 4.0 de C#, actuellement en beta, introduit un nouveau type appelé `dynamic`. Celui ci permet d’accéder à des propriétés ou méthodes d’un objet qui ne sont pas connus statiquement (à la compilation). Ils seront résolus dynamiquement à l’exécution grâce au DLR (Dynamic Language Runtime), qui est l’une des grandes nouveautés de .NET 4.0. Cela permet notamment de faciliter la manipulation d’objets COM, ou de tout autre objet dont on ne connait pas statiquement le type. Pour plus d’informations sur le type `dynamic`, je vous invite à consulter [la documentation MSDN](http://msdn.microsoft.com/en-us/library/dd264736%28VS.100%29.aspx).

En jouant un peu avec la beta de Visual Studio 2010, je me suis rendu compte qu’on pouvait faire des choses très intéressantes avec ce type `dynamic`… En effet, il est possible de créer ses propres objets dynamiques, en contrôlant comment sont évalués les appels aux membres de l’objet. Il faut pour cela implémenter l’interface [`IDynamicMetaObjectProvider`](http://msdn.microsoft.com/en-us/library/system.dynamic.idynamicmetaobjectprovider%28VS.100%29.aspx). Cette interface semble simple à première vue, dans la mesure où elle ne définit qu’un seul membre : la méthode `GetMetaObject`. Là où ça se complique un peu, c’est pour implémenter cette méthode… Il faut construire un `DynamicMetaObject` à partir d’une `Expression`, et j’avoue que je me suis tout d’abord découragé en voyant la complexité de la tâche.

Heureusement, il y a une technique beaucoup plus simple pour implémenter ses propres objets dynamiques : il suffit d’hériter de la classe `DynamicObject`, qui fournit une implementation de base de `IDynamicMetaObjectProvider`. Il suffit ensuite de redéfinir les méthodes qui nous intéressent pour obtenir le comportement souhaité.

Pour le premier exemple, on va s’inspirer du langage Javascript, dans lequel il est possible d’ajouter des membres (propriétés ou méthodes) à un objet existant, de la façon suivante :

```javascript
var x = new Object();
x.Message = "Hello world !";
x.ShowMessage = function()
{
  alert(this.Message);
};
x.ShowMessage();
```

Ce code crée un objet, lui ajoute une propriété `Message` en définissant sa valeur, et ajoute également une méthode `ShowMessage` qui affiche le message.

Dans les précédentes versions de C#, il était impossible de réaliser une telle chose : en effet C# est un langage à typage statique, ce qui implique que l’accès aux membres d’un objet est résolu à la compilation, et non à l’exécution. La classe `Object` n’ayant pas de propriété `Message` ou de méthode `ShowMessage`, on ne peut pas écrire `x.Message` ou `x.ShowMessage()`. Et c’est là qu’entre en jeu le type `dynamic`…

Nous allons donc créer un objet dynamique qui permet d’écrire en C# un code similaire au code Javascript ci-dessus. Pour cela, on va stocker dans un `Dictionary<string, object>` les valeurs des propriétés définies dynamiquement pour l’objet. La clé du fonctionnement de cette classe est de redéfinir les méthodes [`TryGetMember`](http://msdn.microsoft.com/en-us/library/system.dynamic.dynamicobject.trygetmember%28VS.100%29.aspx) et [`TrySetMember`](http://msdn.microsoft.com/en-us/library/system.dynamic.dynamicobject.trygetmember%28VS.100%29.aspx). Ces méthodes implementent la logique permettant de lire ou d’écrire un membre de l’objet dynamique. Pour fixer les idées, voici le code, je le commenterai plus loin.

```csharp
public class MyDynamicObject : DynamicObject
{
    private Dictionary<string, object> _properties = new Dictionary<string, object>();

    public override bool TryGetMember(GetMemberBinder binder, out object result)
    {
        return _properties.TryGetValue(binder.Name, out result);
    }

    public override bool TrySetMember(SetMemberBinder binder, object value)
    {
        _properties[binder.Name] = value;
        return true;
    }
}
```

Détaillons maintenant le code ci-dessus… La méthode `TryGetMember` tente d’obtenir la propriété demandée à partir du dictionnaire. Notez que le nom de la propriété est accessible via la propriété `Name` du paramètre `binder`. Si la propriété existe, sa valeur est renvoyée via le paramètre de sortie `result` et la méthode renvoie `true`. Dans le cas contraire, la méthode renvoie `false`, ce qui cause une exception de type `RuntimeBinderException` à l’endroit où la propriété est accédée en lecture. Cette erreur indique simplement l’échec de la résolution dynamique de la propriété.

La méthode `TrySetMember` effectue l’opération inverse : elle définit la valeur d’une propriété. Si le membre auquel on veut accéder n’existe pas, il est ajouté au dictionnaire, la méthode renvoie donc toujours `true`.

Voyons maintenant comment utiliser cet objet :

```csharp
dynamic x = new MyDynamicObject();
x.Message = "Hello world !";
Console.WriteLine(x.Message);
```

Ce code fonctionne sans problème et affiche “Hello world !” dans la console… sympa, non ?

Et les méthodes dans tout ça ? Et bien, je pourrais vous dire qu’il faut redéfinir la méthode `TryInvokeMember`, qui sert à gérer les appels dynamiques de méthodes… Mais en fait, ce n’est même pas nécessaire ! Notre implémentation gère déjà cette fonctionnalité : il suffit d’affecter un delegate à une propriété de l’objet. Ce ne sera donc pas réellement une méthode membre de l’objet, mais plutôt une propriété qui renvoie un `Delegate` ; mais puisque le code pour invoquer ce delegate est identique à celui de l’appel d’une méthode, on s’en contentera pour l’instant ;). Voilà un exemple d’ajout d’une méthode à l’objet :

```csharp
dynamic x = new MyDynamicObject();
x.Message = "Hello world !";
x.ShowMessage = new Action(
    () =>
    {
        Console.WriteLine(x.Message);
    });
x.ShowMessage();
```

On a donc, à peu de choses près, un code identique au code Javascript qu’on voulait imiter. Et tout ça avec une classe de moins de 10 lignes de code (sans compter les accolades)…

Cette petite classe peut être assez pratique comme objet à tout faire, par exemple pour regrouper des informations sans avoir à créer une classe spécifique. En cela elle est similaire à un type anonyme (déjà présent en C# 3), mais avec l’avantage qu’on peut utiliser l’objet comme valeur de retour d’une méthode, ce qui n’était pas possible avec un type anonyme.

Il est bien sûr possible de créer des objets dynamiques plus utiles que celui-ci… par exemple, voilà un petit wrapper pour un `DataRow`, pour faciliter l'accès aux champs :

```csharp
public class DynamicDataRow : DynamicObject
{
    private DataRow _dataRow;

    public DynamicDataRow(DataRow dataRow)
    {
        if (dataRow == null)
            throw new ArgumentNullException("dataRow");
        this._dataRow = dataRow;
    }

    public DataRow DataRow
    {
        get { return _dataRow; }
    }

    public override bool TryGetMember(GetMemberBinder binder, out object result)
    {
        result = null;
        if (_dataRow.Table.Columns.Contains(binder.Name))
        {
            result = _dataRow[binder.Name];
            return true;
        }
        return false;
    }

    public override bool TrySetMember(SetMemberBinder binder, object value)
    {
        if (_dataRow.Table.Columns.Contains(binder.Name))
        {
            _dataRow[binder.Name] = value;
            return true;
        }
        return false;
    }
}
```

Ajoutons une petite méthode d’extension pour rendre plus naturelle l'utilisation de ce wrapper :

```csharp
public static class DynamicDataRowExtensions
{
    public static dynamic AsDynamic(this DataRow dataRow)
    {
        return new DynamicDataRow(dataRow);
    }
}
```

On peut maintenant écrire un code de ce genre :

```csharp
DataTable table = new DataTable();
table.Columns.Add("FirstName", typeof(string));
table.Columns.Add("LastName", typeof(string));
table.Columns.Add("DateOfBirth", typeof(DateTime));

dynamic row = table.NewRow().AsDynamic();
row.FirstName = "John";
row.LastName = "Doe";
row.DateOfBirth = new DateTime(1981, 9, 12);
table.Rows.Add(row.DataRow);

// Add more rows...
// ...

var bornInThe20thCentury = from r in table.AsEnumerable()
                           let dr = r.AsDynamic()
                           where dr.DateOfBirth.Year > 1900
                           && dr.DateOfBirth.Year <= 2000
                           select new { dr.LastName, dr.FirstName };

foreach (var item in bornInThe20thCentury)
{
    Console.WriteLine("{0} {1}", item.FirstName, item.LastName);
}
```

Voilà, maintenant que vous connaissez le principe, vous pouvez donner libre cours à votre imagination :)
**Mise à jour :** Aussitôt après la publication de cet article, voilà que je tombe sur la classe [`ExpandoObject`](http://msdn.microsoft.com/en-us/library/system.dynamic.expandoobject%28VS.100%29.aspx), qui fait exactement la même chose que la classe `MyDynamicObject` ci-dessus... il semblerait donc que j'ai encore réinventé la roue ! Cela dit, il est toujours intéressant de voir le fonctionnement d'un objet dynamique, ne serait-ce qu'à des fins didactiques ;). Pour plus d'infos sur `ExpandoObject`, voir [ce billet sur le blog C# FAQ](http://blogs.msdn.com/csharpfaq/archive/2009/10/01/dynamic-in-c-4-0-introducing-the-expandoobject.aspx)

