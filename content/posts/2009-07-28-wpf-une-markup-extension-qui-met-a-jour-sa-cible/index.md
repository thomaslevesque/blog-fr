---
layout: post
title: '[WPF] Une markup extension qui met à jour sa cible'
date: 2009-07-28T00:00:00.0000000
url: /2009/07/28/wpf-une-markup-extension-qui-met-a-jour-sa-cible/
tags:
  - markup extension
  - WPF
  - XAML
categories:
  - Code sample
  - WPF
---

Si vous avez lu mes précédents billets sur le sujet, vous savez que je suis un grand fan des markup extensions... Cependant, celles-ci ont une limitation qui peut s'avérer assez gênante : elles ne sont évaluées qu'une seule fois. Il serait pourtant utile de pouvoir les réévaluer pour mettre à jour la propriété cible, comme pour un binding... Cela peut être utile dans différents cas, notamment : 
- si la valeur de la markup extension peut changer en réponse à un évènement
- si l'état de l'objet cible quand la markup extension est évaluée ne permet pas encore de déterminer la valeur à renvoyer, et qu'il faut différer l'évaluation (par exemple si l'on a besoin du DataContext de l'objet, et que celui-ci n'est pas encore défini lors de l'évaluation)

  Voyons donc comment on peut obtenir le comportement voulu...  La méthode `ProvideValue` d'une markup extension prend un paramètre de type `IServiceProvider`, qui fournit, entre autres, un service `IProvideValueTarget`. Cette interface expose des propriétés `TargetObject` et `TargetProperty`, qui permettent d'obtenir l'objet et la propriété cibles de la markup extension. Il est donc possible, si l'on sauvegarde cette information, de mettre à jour la propriété concernée alors que la markup extension a déjà été évaluée.  On va donc créer un classe abstraite `UpdatableMarkupExtension`, qui sauvegarde l'objet et la propriété cible et fournit une méthode pour mettre à jour la valeur :  
```csharp
    public abstract class UpdatableMarkupExtension : MarkupExtension
    {
        private object _targetObject;
        private object _targetProperty;

        protected object TargetObject
        {
            get { return _targetObject; }
        }

        protected object TargetProperty
        {
            get { return _targetProperty; }
        }

        public sealed override object ProvideValue(IServiceProvider serviceProvider)
        {
            IProvideValueTarget target = serviceProvider.GetService(typeof(IProvideValueTarget)) as IProvideValueTarget;
            if (target != null)
            {
                _targetObject = target.TargetObject;
                _targetProperty = target.TargetProperty;
            }

            return ProvideValueInternal(serviceProvider);
        }

        protected void UpdateValue(object value)
        {
            if (_targetObject != null)
            {
                if (_targetProperty is DependencyProperty)
                {
                    DependencyObject obj = _targetObject as DependencyObject;
                    DependencyProperty prop = _targetProperty as DependencyProperty;

                    Action updateAction = () =>  obj.SetValue(prop, value);

                    // Check whether the target object can be accessed from the
                    // current thread, and use Dispatcher.Invoke if it can't

                    if (obj.CheckAccess())
                        updateAction();
                    else
                        obj.Dispatcher.Invoke(updateAction);
                }
                else // _targetProperty is PropertyInfo
                {
                    PropertyInfo prop = _targetProperty as PropertyInfo;
                    prop.SetValue(_targetObject, value, null);
                }
            }
        }

        protected abstract object ProvideValueInternal(IServiceProvider serviceProvider);
    }
```
  Comme il est indispensable de sauvegarder l'objet et la propriété cibles, on marque la méthode `ProvideValue` comme `sealed` pour qu'elle ne puisse pas être redéfinie, et on définit à la place une méthode abstraite `ProvideValueInternal` pour que les classes dérivées puissent fournir leur implémentation.  La méthode `UpdateValue` gère la mise à jour de la propriété cible, qui peut être soit une propriété de dépendance (`DependencyProperty`), soit une propriété CLR classique (`PropertyInfo`). Dans le cas d'une `DependencyProperty`, l'objet cible hérite de `DependencyObject`, et donc de `DispatcherObject` : il faut donc s'assurer qu'on n'accède à cet objet qu'à partir du thread qui le possède, à l'aide des méthodes `CheckAccess` et `Invoke`.  Voyons maintenant comment utiliser cette classe, au travers d'un exemple simple. Supposons qu'on souhaite réaliser une markup extension qui indique si le réseau est disponible, et qui s'utiliserait de la façon suivante :  
```xml
<CheckBox IsChecked="{my:NetworkAvailable}" Content="Network is available" />
```
  On souhaite que la checkbox se mette à jour si l'état de la connexion change (cable branché ou débranché, Wifi hors de portée...). Il faut donc gérer l'évènement `NetworkChange.NetworkAvailabilityChanged`, et mettre à jour la propriété `IsChecked` en conséquence. Notre extension va donc utiliser les fonctionnalités implémentées dans la classe `UpdatableMarkupExtension` :  
```csharp
    public class NetworkAvailableExtension : UpdatableMarkupExtension
    {
        public NetworkAvailableExtension()
        {
            NetworkChange.NetworkAvailabilityChanged += new NetworkAvailabilityChangedEventHandler(NetworkChange_NetworkAvailabilityChanged);
        }

        protected override object ProvideValueInternal(IServiceProvider serviceProvider)
        {
            return NetworkInterface.GetIsNetworkAvailable();
        }

        private void NetworkChange_NetworkAvailabilityChanged(object sender, NetworkAvailabilityEventArgs e)
        {
            UpdateValue(e.IsAvailable);
        }
    }
```
  Notez qu'on s'abonne à l'évènement `NetworkAvailabilityChanged` dans le constructeur de la classe. Si on voulait s'abonner à un évènement de l'objet cible, il faudrait plutôt le faire dans la méthode `ProvideValueInternal`,  pour avoir accès à l'objet cible.  On voit donc qu'il est très simple d'implémenter une markup extension capable de mettre à jour sa cible a posteriori, après la première évaluation. Cela permet d'obtenir un fonctionnement similaire à celui du binding, mais sans être limité aux propriétés de dépendance. J'ai notamment utilisé cette possibilité pour réaliser un système de localisation qui permet de changer la langue "à la volée", sans redémarrer le programme.  **Mise à jour :** En l'état actuel, cette markup extension ne supporte pas l'utilisation dans un template. Pour une explication et une solution à ce problème, lisez [ce billet](http://www.thomaslevesque.fr/2009/08/22/wpf-markup-extensions-et-templates/).

