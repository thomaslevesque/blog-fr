---
layout: post
title: '[WPF] Markup extensions et templates'
date: 2009-08-22T00:00:00.0000000
url: /2009/08/22/wpf-markup-extensions-et-templates/
tags:
  - markup extension
  - template
  - WPF
  - XAML
categories:
  - Code sample
  - WPF
---

*Note : Ce billet est la suite de celui sur [une markup extension qui met à jour sa cible](http://www.thomaslevesque.fr/2009/07/28/wpf-une-markup-extension-qui-met-a-jour-sa-cible/), et réutilise le même code de départ.*  Vous avez peut-être remarqué que l'utilisation d'une markup extension personnalisée dans un template donnait parfois des résultats inattendus... Nous allons voir dans ce billet comment faire une markup extension qui se comporte correctement dans un template.  **Illustration du problème**  Reprenons l'exemple du précédent billet : une markup extension qui renvoie l'état de la connectivité réseau, et met à jour la propriété cible quand le réseau est connecté ou déconnecté :  
```xml
<CheckBox IsChecked="{my:NetworkAvailable}" Content="Network is available" />
```
  Mettons maintenant la même `CheckBox` dans un `ControlTemplate` :  
```xml
<ControlTemplate x:Key="test">
  <CheckBox IsChecked="{my:NetworkAvailable}" Content="Network is available" />
</ControlTemplate>
```
  Et créons un contrôle qui utilise ce template :  
```xml
<Control Template="{StaticResource test}" />
```
  Si on se déconnecte du réseau, on remarque que `la CheckBox` n'est pas automatiquement mise à jour par la `NetworkAvailableExtension`, alors que ça fonctionnait bien quand on l'utilisait hors du template...  **Explication et solution**  La markup expression est évaluée quand elle est rencontrée par le parser XAML : en l'occurrence, lors du parsing du template. Or, à cet instant le contrôle `CheckBox` n'est pas encore créé, la méthode `ProvideValue` ne peut donc pas y accéder... Quand une markup extension est évaluée dans un template, le `TargetObject` est en fait un objet de type `System.Windows.SharedDp`, qui est une classe interne de WPF.  Pour que la markup extension puisse accéder à  sa cible, il faut qu'elle soit évaluée lorsque le template est appliqué : on doit donc retarder son évaluation. Pour y arriver, il suffit en fait de renvoyer l'extension elle-même comme valeur de retour de `ProvideValue` : de cette façon, elle sera de nouveau évaluée lors de la création du contrôle cible.  Pour savoir si l'extension est appelée pour le template ou pour un contrôle "réel", il suffit de tester si le type du `TargetObject` est `System.Windows.SharedDp`. Le code de la méthode `ProvideValue` devient donc :  
```csharp
        public sealed override object ProvideValue(IServiceProvider serviceProvider)
        {
            IProvideValueTarget target = serviceProvider.GetService(typeof(IProvideValueTarget)) as IProvideValueTarget;
            if (target != null)
            {
                if (target.TargetObject.GetType().FullName == "System.Windows.SharedDp")
                    return this;
                _targetObject = target.TargetObject;
                _targetProperty = target.TargetProperty;
            }

            return ProvideValueInternal(serviceProvider);
        }
```
  Voilà, c'est réparé, la `CheckBox` se met à nouveau à jour en cas de changement de connectivité réseau :)  **Encore un os**  Mais ne crions pas victoire trop vite, on n'est pas encore tout à fait au bout de nos peines... Que se passe-t-il si on souhaite maintenant utiliser notre `ControlTemplate` sur plusieurs contrôles ?  
```xml
<Control Template="{StaticResource test}" />
<Control Template="{StaticResource test}" />
```
  Résultat : la seconde checkbox se met à jour, mais pas la première...  La raison est simple : il y a deux contrôles `CheckBox`, mais une seule instance de `NetworkAvailableExtension`, partagée entre toutes les instances du template. Or `NetworkAvailableExtension` ne peut référencer qu'un seul objet cible, c'est donc le dernier pour lequel `ProvideValue` a été appelée qui est conservé...  Il suffit donc de gérer non pas un objet cible, mais une collection d'objets cibles, qui seront tous mis à jour dans la méthode `UpdateValue`. Voilà le code final de la classe de base `UpdatableMarkupExtension` :  
```csharp
    public abstract class UpdatableMarkupExtension : MarkupExtension
    {
        private List<object> _targetObjects = new List<object>();
        private object _targetProperty;

        protected IEnumerable<object> TargetObjects
        {
            get { return _targetObjects; }
        }

        protected object TargetProperty
        {
            get { return _targetProperty; }
        }

        public sealed override object ProvideValue(IServiceProvider serviceProvider)
        {
            // Retrieve target information
            IProvideValueTarget target = serviceProvider.GetService(typeof(IProvideValueTarget)) as IProvideValueTarget;

            if (target != null && target.TargetObject != null)
            {
                // In a template the TargetObject is a SharedDp (internal WPF class)
                // In that case, the markup extension itself is returned to be re-evaluated later
                if (target.TargetObject.GetType().FullName == "System.Windows.SharedDp")
                    return this;

                // Save target information for later updates
                _targetObjects.Add(target.TargetObject);
                _targetProperty = target.TargetProperty;
            }

            // Delegate the work to the derived class
            return ProvideValueInternal(serviceProvider);
        }

        protected virtual void UpdateValue(object value)
        {
            if (_targetObjects.Count > 0)
            {
                // Update the target property of each target object
                foreach (var target in _targetObjects)
                {
                    if (_targetProperty is DependencyProperty)
                    {
                        DependencyObject obj = target as DependencyObject;
                        DependencyProperty prop = _targetProperty as DependencyProperty;

                        Action updateAction = () => obj.SetValue(prop, value);

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
                        prop.SetValue(target, value, null);
                    }
                }
            }
        }

        protected abstract object ProvideValueInternal(IServiceProvider serviceProvider);
    }
```
  La classe `UpdatableMarkupExtension` est donc maintenant pleinement opérationnelle... jusqu'à preuve du contraire ;). Cette classe constitue une bonne base pour toute markup extension devant mettre à jour sa cible, sans avoir à se préoccuper des aspects "bas niveau" du suivi des objets cibles et de leur mise à jour.

