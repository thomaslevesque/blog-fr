---
layout: post
title: '[WPF] Utiliser Linq pour filtrer, trier et grouper les données dans une CollectionView'
date: 2011-11-29T00:00:00.0000000
url: /2011/11/29/wpf-utiliser-linq-pour-filtrer-trier-et-grouper-les-donnes-dans-une-collectionview/
tags:
  - collectionview
  - linq
  - WPF
categories:
  - WPF
---

WPF offre un mécanisme assez simple pour la mise en forme de collections de données, via l’interface `ICollectionView` et ses propriétés `Filter`, `SortDescriptions` et `GroupDescriptions` :  
```csharp
// Collection à laquelle la vue est liée
public ObservableCollection People { get; private set; }
...

// Vue par défaut de la collection People
ICollectionView view = CollectionViewSource.GetDefaultView(People);

// Uniquement les adultes
view.Filter = o => ((Person)o).Age >= 18;

// Tri par nom et prénom
view.SortDescriptions.Add(new SortDescription("LastName", ListSortDirection.Ascending));
view.SortDescriptions.Add(new SortDescription("FirstName", ListSortDirection.Ascending));

// Groupement par pays
view.GroupDescriptions.Add(new PropertyGroupDescription("Country"));
```
  Bien que cette technique ne soit pas très difficile à mettre en œuvre, elle présente certains inconvénients : 
- La syntaxe un peu lourde et pas très naturelle : le fait que le paramètre du filtre soit un `object` alors qu’on sait que les éléments sont de type `Person` réduit la lisibilité, et l’ajout des tris et descriptions comporte beaucoup de répétitions
- Le fait de spécifier les noms des propriétés sous forme de chaine introduit des risques d’erreur, puisqu’ils ne sont pas vérifiés par le compilateur.

 Depuis quelques années, on a pris l'habitude d'utiliser Linq pour faire ce genre de choses… il serait donc pratique de pouvoir le faire aussi pour définir le filtre, le tri et le groupement d'une `ICollectionView`.  Voyons donc quelle syntaxe on pourrait utiliser pour faire ça avec Linq… quelque chose comme ça, par exemple ?  
```csharp
People.Where(p => p.Age >= 18)
      .OrderBy(p => p.LastName)
      .ThenBy(p => p.FirstName)
      .GroupBy(p => p.Country);
```
  Ou encore, en utilisant la syntaxe de requête Linq :  
```csharp
from p in People
where p.Age >= 18
orderby p.LastName, p.FirstName
group p by p.Country;
```
  Bon, évidemment ça ne suffit pas : ce code ne fait rien d’autre que créer une requête Linq sur la collection, il ne modifie pas la `CollectionView`… mais avec un tout petit peu de travail supplémentaire on peut obtenir le résultat voulu :  
```csharp
var query =
    from p in People.ShapeView()
    where p.Age >= 18
    orderby p.LastName, p.FirstName
    group p by p.Country;

query.Apply();
```
  La méthode `ShapeView` renvoie un wrapper qui encapsule la vue par défaut de la collection, et expose des méthodes `Where`, `OrderBy` et `GroupBy` avec les signatures appropriées pour définir la mise en forme. Créer la requête n’a pas d’effet direct, c’est la méthode `Apply` qui permet d’appliquer les changements à la vue : en effet, il vaut mieux tous les appliquer en même temps à l’aide de `ICollectionView.DeferRefresh`, pour ne provoquer un rafraichissement de la vue à chaque nouvelle clause de la requête. Lors de l’appel à `Apply`, on observe que la vue est bien mise à jour pour refléter les clauses de la requête.  Cette solution permet de conserver le typage fort pour le filtrage, le tri et le groupement, avec pour bénéfice immédiat la vérification par le compilateur. C’est également plus concis et plus lisible que le code d’origine… Attention quand même à une chose : certaines requêtes qui seront correctes du point de vue de C# ne seront en fait pas applicables à une `CollectionView`. Par exemple, si vous essayez de grouper par la première lettre du nom (`p.LastName.Substring(0, 1)`), la méthode `GroupBy` échouera, car seules les propriétés sont supportées par `PropertyGroupDescription`.  Notez que le wrapper n’écrase pas les propriétés courantes de la `CollectionView` si vous ne spécifiez pas la clause Linq correspondante, il est donc possible de modifier une vue existante sans devoir tout spécifier à nouveau. Si nécessaire, des méthodes `ClearFilter`, `ClearSort` et `ClearGrouping` permettent de réinitialiser le filtre, le tri et le regroupement :  
```csharp
// Suppression du regroupement et ajout d'un tri :
People.ShapeView()
      .ClearGrouping()
      .OrderBy(p => p.LastName);
      .Apply();
```
  Notez que comme pour une requête Linq “normale”, on peut au choix utiliser la syntaxe de requête ou appeler directement les méthodes, puisqu’il s’agit simplement d’une transformation syntaxique effectuée par le compilateur.  Pour finir, voici le code complet du wrapper et les méthodes d’extension associées :  
```csharp
    public static class CollectionViewShaper
    {
        public static CollectionViewShaper<TSource> ShapeView<TSource>(this IEnumerable<TSource> source)
        {
            var view = CollectionViewSource.GetDefaultView(source);
            return new CollectionViewShaper<TSource>(view);
        }

        public static CollectionViewShaper<TSource> Shape<TSource>(this ICollectionView view)
        {
            return new CollectionViewShaper<TSource>(view);
        }
    }

    public class CollectionViewShaper<TSource>
    {
        private readonly ICollectionView _view;
        private Predicate<object> _filter;
        private readonly List<SortDescription> _sortDescriptions = new List<SortDescription>();
        private readonly List<GroupDescription> _groupDescriptions = new List<GroupDescription>();

        public CollectionViewShaper(ICollectionView view)
        {
            if (view == null)
                throw new ArgumentNullException("view");
            _view = view;
            _filter = view.Filter;
            _sortDescriptions = view.SortDescriptions.ToList();
            _groupDescriptions = view.GroupDescriptions.ToList();
        }

        public void Apply()
        {
            using (_view.DeferRefresh())
            {
                _view.Filter = _filter;
                _view.SortDescriptions.Clear();
                foreach (var s in _sortDescriptions)
                {
                    _view.SortDescriptions.Add(s);
                }
                _view.GroupDescriptions.Clear();
                foreach (var g in _groupDescriptions)
                {
                    _view.GroupDescriptions.Add(g);
                }
            }
        }
            
        public CollectionViewShaper<TSource> ClearGrouping()
        {
            _groupDescriptions.Clear();
            return this;
        }

        public CollectionViewShaper<TSource> ClearSort()
        {
            _sortDescriptions.Clear();
            return this;
        }

        public CollectionViewShaper<TSource> ClearFilter()
        {
            _filter = null;
            return this;
        }

        public CollectionViewShaper<TSource> ClearAll()
        {
            _filter = null;
            _sortDescriptions.Clear();
            _groupDescriptions.Clear();
            return this;
        }

        public CollectionViewShaper<TSource> Where(Func<TSource, bool> predicate)
        {
            _filter = o => predicate((TSource)o);
            return this;
        }

        public CollectionViewShaper<TSource> OrderBy<TKey>(Expression<Func<TSource, TKey>> keySelector)
        {
            return OrderBy(keySelector, true, ListSortDirection.Ascending);
        }

        public CollectionViewShaper<TSource> OrderByDescending<TKey>(Expression<Func<TSource, TKey>> keySelector)
        {
            return OrderBy(keySelector, true, ListSortDirection.Descending);
        }

        public CollectionViewShaper<TSource> ThenBy<TKey>(Expression<Func<TSource, TKey>> keySelector)
        {
            return OrderBy(keySelector, false, ListSortDirection.Ascending);
        }

        public CollectionViewShaper<TSource> ThenByDescending<TKey>(Expression<Func<TSource, TKey>> keySelector)
        {
            return OrderBy(keySelector, false, ListSortDirection.Descending);
        }

        private CollectionViewShaper<TSource> OrderBy<TKey>(Expression<Func<TSource, TKey>> keySelector, bool clear, ListSortDirection direction)
        {
            string path = GetPropertyPath(keySelector.Body);
            if (clear)
                _sortDescriptions.Clear();
            _sortDescriptions.Add(new SortDescription(path, direction));
            return this;
        }

        public CollectionViewShaper<TSource> GroupBy<TKey>(Expression<Func<TSource, TKey>> keySelector)
        {
            string path = GetPropertyPath(keySelector.Body);
            _groupDescriptions.Add(new PropertyGroupDescription(path));
            return this;
        }

        private static string GetPropertyPath(Expression expression)
        {
            var names = new Stack<string>();
            var expr = expression;
            while (expr != null && !(expr is ParameterExpression) && !(expr is ConstantExpression))
            {
                var memberExpr = expr as MemberExpression;
                if (memberExpr == null)
                    throw new ArgumentException("The selector body must contain only property or field access expressions");
                names.Push(memberExpr.Member.Name);
                expr = memberExpr.Expression;
            }
            return String.Join(".", names.ToArray());
        }
    }
```

