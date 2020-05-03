---
layout: post
title: '[C#] Relation parent/enfant et sérialisation XML'
date: 2009-06-11T00:00:00.0000000
url: /2009/06/11/c-relation-parentenfant-et-serialisation-xml/
tags:
  - C#
  - collection
  - parent/enfant
  - sérialisation XML
categories:
  - Astuces
  - Code sample
---

Me revoilà avec un peu de retard, j'ai un peu manqué de temps libre ces dernières semaines... Voilà donc un petit post pour présenter une idée qui m'est venue récemment. Pour une fois, il ne sera pas question de WPF, c'est de conception C# qu'il s'agit !  **Le problème**  Il est assez courant, dans un programme, d'avoir un objet parent qui possède une collection d'enfants ayant une référence vers leur parent. C'est le cas, par exemple, des contrôles Windows Forms, qui possèdent une collection de contrôles enfants (`Controls`), et une propriété qui indique le contrôle parent (`Parent`).  Ce type de structure est assez simple à réaliser, cela nécessite juste un peu de code pour maintenir la cohérence de la relation. Là où ça se complique un peu, c'est quand on veut sérialiser l'objet parent en XML... Prenons un exemple simple (purement théorique bien sûr) :  
```csharp
    public class Parent
    {
        public Parent()
        {
            this.Children = new List<Child>();
        }

        public string Name { get; set; }

        public List<Child> Children { get; set; }

        public void AddChild(Child child)
        {
            child.ParentObject = this;
            this.Children.Add(child);
        }

        public void RemoveChild(Child child)
        {
            this.Children.Remove(child);
            child.ParentObject = null;
        }
    }
```

```csharp
    public class Child
    {
        public string Name { get; set; }

        public Parent ParentObject { get; set; }
    }
```
  Créons une instance de `Parent` avec quelques enfants, et essayons de la sérialiser en XML :  
```csharp
            Parent p = new Parent { Name = "The parent" };
            p.AddChild(new Child { Name = "First child" });
            p.AddChild(new Child { Name = "Second child" });

            string xml;
            XmlSerializer xs = new XmlSerializer(typeof(Parent));
            using (StringWriter wr = new StringWriter())
            {
                xs.Serialize(wr, p);
                xml = wr.ToString();
            }

            Console.WriteLine(xml);
```
  Quand on sérialise l'objet `Parent`, il se produit une `InvalidOperationException` due à une référence circulaire : en effet, le parent référence les enfants, qui référencent le parent, qui référence les enfants... et ainsi de suite. La première solution qui vient à l'esprit est de ne pas sérialiser la propriété `Child.ParentObject` : il suffit pour cela de lui appliquer l'attribut `XmlIgnore`. La sérialisation se passe alors sans problème, mais quand on désérialise, mauvaise surprise : la propriété `ParentObject` n'est pas renseignée, et pour cause, puisqu'elle n'a pas été sauvegardée...  On pourrait opter pour une solution simple : après la désérialisation : parcourir la liste des enfants pour affecter manuellement la propriété `ParentObject`. Mais ce n'est vraiment pas très élégant... et comme j'aime bien écrire du beau code, j'ai donc cherché autre chose ;)  **La solution**  La solution qui m'est venue à l'esprit consiste en une collection générique spécialisée `ChildItemCollection<P,T>`, et une interface `IChildItem<P>` que les enfants doivent implémenter.  L'interface `IChildItem<P>` définit simplement une propriété `Parent`, de type `P` :  
```csharp
    /// <summary>
    /// Defines the contract for an object that has a parent object
    /// </summary>
    /// <typeparam name="P">Type of the parent object</typeparam>
    public interface IChildItem<P> where P : class
    {
        P Parent { get; set; }
    }
```
  La collection `ChildItemCollection<P,T>` implémente `IList<T>` en délégant l'implémentation à une `List<T>` ou à la collection passée en paramètre du constructeur, et gère le maintien de la relation parent/enfant :  
```csharp
    /// <summary>
    /// Collection of child items. This collection automatically set the
    /// Parent property of the child items when they are added or removed
    /// </summary>
    /// <typeparam name="P">Type of the parent object</typeparam>
    /// <typeparam name="T">Type of the child items</typeparam>
    public class ChildItemCollection<P, T> : IList<T>
        where P : class
        where T : IChildItem<P>
    {
        private P _parent;
        private IList<T> _collection;

        public ChildItemCollection(P parent)
        {
            this._parent = parent;
            this._collection = new List<T>();
        }

        public ChildItemCollection(P parent, IList<T> collection)
        {
            this._parent = parent;
            this._collection = collection;
        }

        #region IList<T> Members

        public int IndexOf(T item)
        {
            return _collection.IndexOf(item);
        }

        public void Insert(int index, T item)
        {
            if (item != null)
                item.Parent = _parent;
            _collection.Insert(index, item);
        }

        public void RemoveAt(int index)
        {
            T oldItem = _collection[index];
            _collection.RemoveAt(index);
            if (oldItem != null)
                oldItem.Parent = null;
        }

        public T this[int index]
        {
            get
            {
                return _collection[index];
            }
            set
            {
                T oldItem = _collection[index];
                if (value != null)
                    value.Parent = _parent;
                _collection[index] = value;
                if (oldItem != null)
                    oldItem.Parent = null;
            }
        }

        #endregion

        #region ICollection<T> Members

        public void Add(T item)
        {
            if (item != null)
                item.Parent = _parent;
            _collection.Add(item);
        }

        public void Clear()
        {
            foreach (T item in _collection)
            {
                if (item != null)
                    item.Parent = null;
            }
            _collection.Clear();
        }

        public bool Contains(T item)
        {
            return _collection.Contains(item);
        }

        public void CopyTo(T[] array, int arrayIndex)
        {
            _collection.CopyTo(array, arrayIndex);
        }

        public int Count
        {
            get { return _collection.Count; }
        }

        public bool IsReadOnly
        {
            get { return _collection.IsReadOnly; }
        }

        public bool Remove(T item)
        {
            bool b = _collection.Remove(item);
            if (item != null)
                item.Parent = null;
            return b;
        }

        #endregion

        #region IEnumerable<T> Members

        public IEnumerator<T> GetEnumerator()
        {
            return _collection.GetEnumerator();
        }

        #endregion

        #region IEnumerable Members

        System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator()
        {
            return (_collection as System.Collections.IEnumerable).GetEnumerator();
        }

        #endregion
    }
```
  Voyons donc comment utiliser cette solution dans notre exemple précédent... Nous allons d'abord modifier la classe `Child` pour qu'elle implémente l'interface `IChildItem<Parent>` :  
```csharp
    public class Child : IChildItem<Parent>
    {
        public string Name { get; set; }

        [XmlIgnore]
        public Parent ParentObject { get; internal set; }

        #region IChildItem<Parent> Members

        Parent IChildItem<Parent>.Parent
        {
            get
            {
                return this.ParentObject;
            }
            set
            {
                this.ParentObject = value;
            }
        }

        #endregion
    }
```
  Vous remarquerez qu'on implémente l'interface `IChildItem<Parent>` de façon *explicite* : l'intérêt est de "masquer" la propriété `Parent`, qui ne sera accessible que si on manipule l'objet `Child` via une variable de type `IChildItem<Parent>`. On définit aussi l'accesseur `set` de la propriété `ParentObject` comme étant `internal`, de façon à empêcher sa modification à partir d'un autre assembly.  Dans la classe `Parent`, il suffit de remplacer la `List<Child>` par une `ChildItemCollection<Parent, Child>`. On supprime au passage les méthodes `AddChild` et `RemoveChild`, qui ne sont plus nécessaires puisque la classe `ChildItemCollection<P,T>` se charge de renseigner la propriété `Parent`.  
```csharp
    public class Parent
    {
        public Parent()
        {
            this.Children = new ChildItemCollection<Parent, Child>(this);
        }

        public string Name { get; set; }

        public ChildItemCollection<Parent, Child> Children { get; private set; }
    }
```
  Notez qu'on passe au constructeur de `ChildItemCollection<Parent, Child>` une référence vers l'objet courant : c'est de cette façon que la collection sait quel doit être le parent de ses éléments.  Le code utilisé précédemment pour sérialiser un `Parent` fonctionne maintenant correctement. Lors de la désérialisation, la propriété `Child.ParentObject` n'est pas affectée quand le `Child` est désérialisé (puisqu'elle a l'attribut `XmlIgnore`), mais lors de l'ajout à la collection `Parent.Children`.  Cette solution permet donc de conserver la relation parent/enfant lors de la sérialisation XML, sans recourir à des bricolages peu élégants... Notez cependant une restriction : si la propriété `ParentObject` est modifiée autrement que par la classe `ChildItemCollection<P,T>`, la cohérence de la relation parent/enfant est rompue. Il est cependant assez simple d'ajouter à l'accesseur `set` la logique pour maintenir la cohérence, je ne l'ai pas fait dans cet exemple par souci de clarté et de simplicité.

