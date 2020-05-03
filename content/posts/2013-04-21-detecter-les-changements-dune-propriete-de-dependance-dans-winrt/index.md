---
layout: post
title: Détecter les changements d’une propriété de dépendance dans WinRT
date: 2013-04-21T00:00:00.0000000
url: /2013/04/21/detecter-les-changements-dune-propriete-de-dependance-dans-winrt/
tags:
  - binding
  - C#
  - dependency property
  - winrt
  - XAML
categories:
  - Astuces
  - WinRT
---


Aujourd'hui j'aimerais partager une astuce que j'ai utilisée en développant ma première application Windows Store. Je suis complètement nouveau sur cette technologie et c'est mon premier billet à ce sujet, donc j'espère que je ne vais pas trop me ridiculiser...

Il est souvent utile d'être notifié quand la valeur d'une propriété de dépendance change ; beaucoup de contrôles exposent des évènements à cet effet, mais ce n'est pas toujours le cas. Par exemple, récemment j'essayais de détecter les changements de la propriété `Content` d'un `ContentControl`. En WPF, j'aurais utilisé la classe [`DependencyPropertyDescriptor`](http://msdn.microsoft.com/en-us/library/system.componentmodel.dependencypropertydescriptor.aspx), mais elle n'est pas disponible dans WinRT.

Heureusement, il y a un mécanisme qui existe sur toutes les plateformes XAML, et qui peut résoudre ce problème: le binding. La solution est donc simplement de créer une classe avec un propriété "bidon" qui est liée à la propriété qu'on souhaite observer, et d'appeler un handler quand la valeur de cette propriété bidon change. Pour rendre ça un peu plus propre et masquer l'implémentation réelle, j'ai emballé ça sous forme d'une méthode d'extension qui renvoie un `IDisposable`:

```
    public static class DependencyObjectExtensions
    {
        public static IDisposable WatchProperty(this DependencyObject target,
                                                string propertyPath,
                                                DependencyPropertyChangedEventHandler handler)
        {
            return new DependencyPropertyWatcher(target, propertyPath, handler);
        }

        class DependencyPropertyWatcher : DependencyObject, IDisposable
        {
            private DependencyPropertyChangedEventHandler _handler;

            public DependencyPropertyWatcher(DependencyObject target,
                                             string propertyPath,
                                             DependencyPropertyChangedEventHandler handler)
            {
                if (target == null) throw new ArgumentNullException("target");
                if (propertyPath == null) throw new ArgumentNullException("propertyPath");
                if (handler == null) throw new ArgumentNullException("handler");

                _handler = handler;

                var binding = new Binding
                {
                    Source = target,
                    Path = new PropertyPath(propertyPath),
                    Mode = BindingMode.OneWay
                };
                BindingOperations.SetBinding(this, ValueProperty, binding);
            }

            private static readonly DependencyProperty ValueProperty =
                DependencyProperty.Register(
                    "Value",
                    typeof(object),
                    typeof(DependencyPropertyWatcher),
                    new PropertyMetadata(null, ValuePropertyChanged));

            private static void ValuePropertyChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
            {
                var watcher = d as DependencyPropertyWatcher;
                if (watcher == null)
                    return;

                watcher.OnValueChanged(e);
            }

            private void OnValueChanged(DependencyPropertyChangedEventArgs e)
            {
                var handler = _handler;
                if (handler != null)
                    handler(this, e);
            }

            public void Dispose()
            {
                _handler = null;
                // There is no ClearBinding method, so set a dummy binding instead
                BindingOperations.SetBinding(this, ValueProperty, new Binding());
            }
        }
    }
```

On peut l'utiliser comme ceci:

```
// Abonnement
watcher = myControl.WatchProperty("Content", myControl_ContentChanged);

// Désabonnement
watcher.Dispose();
```

J'espère que vous trouverez cela utile!

