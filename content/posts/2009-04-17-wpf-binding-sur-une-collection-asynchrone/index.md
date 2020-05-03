---
layout: post
title: '[WPF] Binding sur une collection asynchrone'
date: 2009-04-17T00:00:00.0000000
url: /2009/04/17/wpf-binding-sur-une-collection-asynchrone/
tags:
  - asynchrone
  - binding
  - collection
  - MVVM
  - WPF
categories:
  - Code sample
  - WPF
---

Comme je l'avais évoqué dans mon précédent post, on ne peut pas ajouter des éléments à une `ObservableCollection` à partir d'un autre thread si une vue est bindée sur la collection : cela provoque une `NotSupportedException`. Prenons l'exemple d'une `ListBox` bindée sur une collection de chaines de caractères appartenant au *ViewModel* :  
```csharp
        private ObservableCollection<string> _strings = new ObservableCollection<string>();
        public ObservableCollection<string> Strings
        {
            get { return _strings; }
            set
            {
                _strings = value;
                OnPropertyChanged("Strings");
            }
        }
```

```xml
<ListBox ItemsSource="{Binding Strings}"/>
```
  Si on ajoute des éléments à cette collection hors du thread principal, on obtient l'exception citée plus haut. Une solution est de créer une nouvelle liste, puis de l'affecter à la propriété `Strings` quand elle est remplie, mais dans ce cas l'interface graphique ne reflète pas la progression : les éléments de la liste apparaissent tous à la fois quand la liste est remplie, et non au fur et à mesure que les éléments sont ajoutés. Si la liste correspond aux résultats d'une recherche, par exemple, ça peut être assez gênant car l'utilisateur s'attend à voir les résultats apparaître au fur et à mesure qu'ils sont trouvés (comme dans la recherche Windows).  Un moyen simple d'obtenir le comportement voulu est de créer une classe héritée de `ObservableCollection` qui déclenche les évènements `CollectionChanged` et `PropertyChanged` sur le thread principal au lieu du thread courant. La classe `AsyncOperation` se prête parfaitement à cet objectif : elle permet de "poster" un évènement sur le thread qui l'a créée. Elle est notamment utilisée par le composant `BackgroundWorker` et de nombreuses méthodes asynchrones du framework (`PictureBox.LoadAsync`, `WebClient.DownloadAsync`, etc...).  Voici donc le code d'une collection `AsyncObservableCollection` qui peut être modifiée à partir de n'importe quel thread tout en notifiant l'interface graphique lors d'une modification :  
```csharp
    public class AsyncObservableCollection<T> : ObservableCollection<T>
    {
        private AsyncOperation asyncOp = null;

        public AsyncObservableCollection()
        {
            CreateAsyncOp();
        }

        public AsyncObservableCollection(IEnumerable<T> list)
            : base(list)
        {
            CreateAsyncOp();
        }

        private void CreateAsyncOp()
        {
            // Create the AsyncOperation to post events on the creator thread
            asyncOp = AsyncOperationManager.CreateOperation(null);
        }

        protected override void OnCollectionChanged(NotifyCollectionChangedEventArgs e)
        {
            // Post the CollectionChanged event on the creator thread
            asyncOp.Post(RaiseCollectionChanged, e);
        }

        private void RaiseCollectionChanged(object param)
        {
            // We are in the creator thread, call the base implementation directly
           base.OnCollectionChanged((NotifyCollectionChangedEventArgs)param);
        }

        protected override void OnPropertyChanged(PropertyChangedEventArgs e)
        {
            // Post the PropertyChanged event on the creator thread
            asyncOp.Post(RaisePropertyChanged, e);
        }

        private void RaisePropertyChanged(object param)
        {
            // We are in the creator thread, call the base implementation directly
            base.OnPropertyChanged((PropertyChangedEventArgs)param);
        }
    }
```
  La seule contrainte est de créer les instances de cette collection sur le thread de l'interface graphique, afin que les évènements soient bien déclenchés sur ce thread.  Si on reprend le code de l'exemple précédent, la seule chose à changer pour pouvoir modifier la collection à partir d'un autre thread est l'instantiation de la collection dans le *ViewModel* :  
```csharp
private ObservableCollection<string> _strings = new AsyncObservableCollection<string>();
```
  La `ListBox` peut maintenant refléter en temps réel les changements intervenus dans la collection.  Enjoy ;)  **Mise à jour :** Je viens de remarquer un bug dans mon implémentation : dans certains cas le fait de passer par un `Post` pour lever l'évènement alors que la collection est modifiée à partir du thread principal peut produire un comportement inattendu. Dans ce cas il faut évidemment lever l'évènement directement, en vérifiant que le `SynchronizationContext` courant est le même que celui dans lequel a été créée la collection. Et puisqu'on en est à se préoccuper du `SynchronizationContext`, autant l'utiliser directement et se passer de l'`AsyncOperation`, qui finalement n'apporte rien. Voici donc la nouvelle implémentation :  
```csharp
    public class AsyncObservableCollection<T> : ObservableCollection<T>
    {
        private SynchronizationContext _synchronizationContext = SynchronizationContext.Current;

        public AsyncObservableCollection()
        {
        }

        public AsyncObservableCollection(IEnumerable<T> list)
            : base(list)
        {
        }

        protected override void OnCollectionChanged(NotifyCollectionChangedEventArgs e)
        {
            if (SynchronizationContext.Current == _synchronizationContext)
            {
                // Raise the CollectionChanged event on the current thread
                RaiseCollectionChanged(e);
            }
            else
            {
                // Raise the CollectionChanged event on the creator thread
                _synchronizationContext.Send(RaiseCollectionChanged, e);
            }
        }

        private void RaiseCollectionChanged(object param)
        {
            // We are in the creator thread, call the base implementation directly
            base.OnCollectionChanged((NotifyCollectionChangedEventArgs)param);
        }

        protected override void OnPropertyChanged(PropertyChangedEventArgs e)
        {
            if (SynchronizationContext.Current == _synchronizationContext)
            {
                // Raise the PropertyChanged event on the current thread
                RaisePropertyChanged(e);
            }
            else
            {
                // Raise the PropertyChanged event on the creator thread
                _synchronizationContext.Send(RaisePropertyChanged, e);
            }
        }

        private void RaisePropertyChanged(object param)
        {
            // We are in the creator thread, call the base implementation directly
            base.OnPropertyChanged((PropertyChangedEventArgs)param);
        }
    }
```
**Mise à jour :** modifié le code pour utiliser `Send` plutôt que `Post`. L'utilisation de `Post` faisait que l'évènement était déclenché de façon *asynchrone* sur le thread UI, ce qui pouvait causer une *race condition* si la collection était modifiée à nouveau avant que l'évènement ne soit géré.

