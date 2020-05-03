---
layout: post
title: '[WPF 4.5] Abonnement à un évènement à l’aide d’une markup extension'
date: 2011-09-23T00:00:00.0000000
url: /2011/09/23/wpf-4-5-abonnement-a-un-evenement-a-laide-dune-markup-extension/
tags:
  - .NET 4.5
  - évènements
  - markup extension
  - WPF
  - XAML
categories:
  - WPF
---

Voilà un certain temps que je n'avais plus parlé des markup extensions... J'y reviens à l'occasion de la sortie de Visual Studio 11 Developer Preview, qui introduit un certain nombre de [nouveautés](http://msdn.microsoft.com/en-us/library/bb613588%28v=VS.110%29.aspx) dans WPF. La nouveauté dont je vais parler n'est sans doute pas la plus spectaculaire, mais elle vient combler un manque des versions précédentes : le support des markup extensions pour les évènements.  Jusqu'ici, il était possible d'utiliser une markup extension en XAML pour affecter une valeur à une propriété, mais on ne pouvait pas faire la même chose pour s'abonner à un évènement. Dans WPF 4.5, c'est désormais possible. Voilà donc un petit exemple de ce que cela permet de faire...  Quand on utilise le pattern MVVM, on associe souvent des commandes du ViewModel à des contrôles de la vue, via le mécanisme de binding. Cette approche fonctionne généralement assez bien, mais elle présente certains inconvénients : 
- cela introduit beaucoup de code de "plomberie" dans le ViewModel
- tous les contrôles n'ont pas une propriété `Command` (en fait, la plupart ne l'ont pas), et quand cette propriété existe, elle ne correspond qu'à un seul évènement du contrôle (par exemple le clic sur un bouton). Il n'y a pas de moyen vraiment simple de relier les autres évènements à des commandes du ViewModel.

  Il serait plus pratique de pouvoir lier directement l'évènement à une méthode du ViewModel de la façon suivante: 
```xml
        <Button Content="Click me"
                Click="{my:EventBinding OnClick}" />
```
 Avec la méthode `OnClick` définie dans le ViewModel: 
```csharp
        public void OnClick(object sender, EventArgs e)
        {
            MessageBox.Show("Hello world!");
        }
```
  Eh bien cela est désormais possible ! Voilà donc une petite preuve de concept... La classe `EventBindingExtension` présentée ci-dessous obtient d'abord le `DataContext` du contrôle, puis recherche la méthode spécifiée dans le `DataContext`, et renvoie un delegate pour cette méthode:  
```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Diagnostics;
using System.Linq;
using System.Reflection;
using System.Windows;
using System.Windows.Markup;


    public class EventBindingExtension : MarkupExtension
    {
        public EventBindingExtension() { }

        public EventBindingExtension(string eventHandlerName)
        {
            this.EventHandlerName = eventHandlerName;
        }

        [ConstructorArgument("eventHandlerName")]
        public string EventHandlerName { get; set; }

        public override object ProvideValue(IServiceProvider serviceProvider)
        {
            if (string.IsNullOrEmpty(EventHandlerName))
                throw new ArgumentException("The EventHandlerName property is not set", "EventHandlerName");

            var target = (IProvideValueTarget)serviceProvider.GetService(typeof(IProvideValueTarget));

            EventInfo eventInfo = target.TargetProperty as EventInfo;
            if (eventInfo == null)
                throw new InvalidOperationException("The target property must be an event");
            
            object dataContext = GetDataContext(target.TargetObject);
            if (dataContext == null)
                throw new InvalidOperationException("No DataContext found");

            var handler = GetHandler(dataContext, eventInfo, EventHandlerName);
            if (handler == null)
                throw new ArgumentException("No valid event handler was found", "EventHandlerName");

            return handler;
        }

        #region Helper methods

        static object GetHandler(object dataContext, EventInfo eventInfo, string eventHandlerName)
        {
            Type dcType = dataContext.GetType();

            var method = dcType.GetMethod(
                eventHandlerName,
                GetParameterTypes(eventInfo));
            if (method != null)
            {
                if (method.IsStatic)
                    return Delegate.CreateDelegate(eventInfo.EventHandlerType, method);
                else
                    return Delegate.CreateDelegate(eventInfo.EventHandlerType, dataContext, method);
            }

            return null;
        }

        static Type[] GetParameterTypes(EventInfo eventInfo)
        {
            var invokeMethod = eventInfo.EventHandlerType.GetMethod("Invoke");
            return invokeMethod.GetParameters().Select(p => p.ParameterType).ToArray();
        }

        static object GetDataContext(object target)
        {
            var depObj = target as DependencyObject;
            if (depObj == null)
                return null;

            return depObj.GetValue(FrameworkElement.DataContextProperty)
                ?? depObj.GetValue(FrameworkContentElement.DataContextProperty);
        }

        #endregion
    }
```
  Cette classe est utilisable comme dans l'exemple présenté plus haut.  En l'état, cette markup extension présente une limitation un peu gênante : le `DataContext` doit être défini avant l'appel à `ProvideValue`, sinon il n'est pas possible de trouver la méthode qui gère l'évènement. Une solution pourrait être de s'abonner à l'évènement `DataContextChanged` pour s'abonner à l'évènement plus tard, mais en attendant il faut quand même renvoyer une valeur... et si on renvoie `null`, on obtient une exception car on ne peut pas s'abonner à un évènement avec un handler `null`. Il faudrait donc renvoyer un handler "bidon" généré dynamiquement en fonction de la signature de l'évènement. Voilà qui complique un peu les choses... mais ça reste faisable.  Voici une deuxième version qui implémente cette amélioration :  
```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Diagnostics;
using System.Linq;
using System.Reflection;
using System.Reflection.Emit;
using System.Windows;
using System.Windows.Markup;

    public class EventBindingExtension : MarkupExtension
    {
        private EventInfo _eventInfo;

        public EventBindingExtension() { }

        public EventBindingExtension(string eventHandlerName)
        {
            this.EventHandlerName = eventHandlerName;
        }

        [ConstructorArgument("eventHandlerName")]
        public string EventHandlerName { get; set; }

        public override object ProvideValue(IServiceProvider serviceProvider)
        {
            if (string.IsNullOrEmpty(EventHandlerName))
                throw new ArgumentException("The EventHandlerName property is not set", "EventHandlerName");

            var target = (IProvideValueTarget)serviceProvider.GetService(typeof(IProvideValueTarget));

            var targetObj = target.TargetObject as DependencyObject;
            if (targetObj == null)
                throw new InvalidOperationException("The target object must be a DependencyObject");

            _eventInfo = target.TargetProperty as EventInfo;
            if (_eventInfo == null)
                throw new InvalidOperationException("The target property must be an event");

            object dataContext = GetDataContext(targetObj);
            if (dataContext == null)
            {
                SubscribeToDataContextChanged(targetObj);
                return GetDummyHandler(_eventInfo.EventHandlerType);
            }

            var handler = GetHandler(dataContext, _eventInfo, EventHandlerName);
            if (handler == null)
            {
                Trace.TraceError(
                    "EventBinding: no suitable method named '{0}' found in type '{1}' to handle event '{2'}",
                    EventHandlerName,
                    dataContext.GetType(),
                    _eventInfo);
                return GetDummyHandler(_eventInfo.EventHandlerType);
            }

            return handler;
            
        }

        #region Helper methods

        static Delegate GetHandler(object dataContext, EventInfo eventInfo, string eventHandlerName)
        {
            Type dcType = dataContext.GetType();

            var method = dcType.GetMethod(
                eventHandlerName,
                GetParameterTypes(eventInfo.EventHandlerType));
            if (method != null)
            {
                if (method.IsStatic)
                    return Delegate.CreateDelegate(eventInfo.EventHandlerType, method);
                else
                    return Delegate.CreateDelegate(eventInfo.EventHandlerType, dataContext, method);
            }

            return null;
        }

        static Type[] GetParameterTypes(Type delegateType)
        {
            var invokeMethod = delegateType.GetMethod("Invoke");
            return invokeMethod.GetParameters().Select(p => p.ParameterType).ToArray();
        }

        static object GetDataContext(DependencyObject target)
        {
            return target.GetValue(FrameworkElement.DataContextProperty)
                ?? target.GetValue(FrameworkContentElement.DataContextProperty);
        }

        static readonly Dictionary<Type, Delegate> _dummyHandlers = new Dictionary<Type, Delegate>();

        static Delegate GetDummyHandler(Type eventHandlerType)
        {
            Delegate handler;
            if (!_dummyHandlers.TryGetValue(eventHandlerType, out handler))
            {
                handler = CreateDummyHandler(eventHandlerType);
                _dummyHandlers[eventHandlerType] = handler;
            }
            return handler;
        }

        static Delegate CreateDummyHandler(Type eventHandlerType)
        {
            var parameterTypes = GetParameterTypes(eventHandlerType);
            var returnType = eventHandlerType.GetMethod("Invoke").ReturnType;
            var dm = new DynamicMethod("DummyHandler", returnType, parameterTypes);
            var il = dm.GetILGenerator();
            if (returnType != typeof(void))
            {
                if (returnType.IsValueType)
                {
                    var local = il.DeclareLocal(returnType);
                    il.Emit(OpCodes.Ldloca_S, local);
                    il.Emit(OpCodes.Initobj, returnType);
                    il.Emit(OpCodes.Ldloc_0);
                }
                else
                {
                    il.Emit(OpCodes.Ldnull);
                }
            }
            il.Emit(OpCodes.Ret);
            return dm.CreateDelegate(eventHandlerType);
        }

        private void SubscribeToDataContextChanged(DependencyObject targetObj)
        {
            DependencyPropertyDescriptor
                .FromProperty(FrameworkElement.DataContextProperty, targetObj.GetType())
                .AddValueChanged(targetObj, TargetObject_DataContextChanged);
        }

        private void UnsubscribeFromDataContextChanged(DependencyObject targetObj)
        {
            DependencyPropertyDescriptor
                .FromProperty(FrameworkElement.DataContextProperty, targetObj.GetType())
                .RemoveValueChanged(targetObj, TargetObject_DataContextChanged);
        }

        private void TargetObject_DataContextChanged(object sender, EventArgs e)
        {
            DependencyObject targetObj = sender as DependencyObject;
            if (targetObj == null)
                return;

            object dataContext = GetDataContext(targetObj);
            if (dataContext == null)
                return;

            var handler = GetHandler(dataContext, _eventInfo, EventHandlerName);
            if (handler != null)
            {
                _eventInfo.AddEventHandler(targetObj, handler);
            }
            UnsubscribeFromDataContextChanged(targetObj);
        }

        #endregion
    }
```
  Voilà donc un exemple du genre de choses qu'on peut faire grâce à cette nouvelle fonctionnalité de WPF. On pourrait aussi imaginer un système de "behavior" similaire à ce qu'on peut faire avec des propriétés attachées, par exemple pour réaliser une action standard lorsqu'un évènement se produit. Les possiblités sont sans doute nombreuses, je vous laisse le soin de les trouver ;)

