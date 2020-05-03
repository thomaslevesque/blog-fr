---
layout: post
title: '[WPF] Utiliser les InputBindings avec le pattern MVVM'
date: 2009-03-17T00:00:00.0000000
url: /2009/03/17/wpf-utiliser-les-inputbindings-avec-le-pattern-mvvm/
tags:
  - InputBinding
  - KeyBinding
  - markup extension
  - MVVM
  - WPF
  - XAML
categories:
  - Code sample
  - WPF
---

Si vous développez des applications WPF en suivant le design pattern Model-View-ViewModel, vous vous êtes peut-être déjà trouvé confronté au problème suivant : comment, en XAML, lier un raccourci clavier ou une action de la souris à une commande du ViewModel ? Idéalement, on aimerait pouvoir faire comme ça :  
```xml
    <UserControl.InputBindings>
        <KeyBinding Modifiers="Control" Key="E" Command="{Binding EditCommand}"/>
    </UserControl.InputBindings>
```
  Malheureusement, ce code ne fonctionne pas, pour deux raisons : 
1. La propriété Command n'est pas une DependencyProperty, on ne peut donc pas faire de binding dessus
2. Les InputBindings ne font pas partie de l'arbre logique ou visuel du contrôle, ils n'héritent donc pas du DataContext

  Une solution, bien sûr, serait de passer par le code-behind pour créer les InputBindings, mais en général, dans une application développée selon le pattern MVVM, on préfère éviter d'écrire du code-behind. J'ai longuement cherché des solutions alternatives pour pouvoir le faire en XAML, mais la plupart sont relativement complexes et peu intuitives. J'ai donc finalement créé une markup extension qui permet de se binder à une commande du DataContext, à n'importe quel endroit du code XAML, même si l'élément n'hérite pas du DataContext.  Cette extension s'utilise comme un simple binding :  
```xml
    <UserControl.InputBindings>
        <KeyBinding Modifiers="Control" Key="E" Command="{input:CommandBinding EditCommand}"/>
    </UserControl.InputBindings>
```
  (Le namespace XML *input* étant mappé sur le namespace CLR où est déclarée la markup extension)  Pour réaliser cette extension, j'avoue que j'ai un peu triché... j'ai fouillé avec Reflector le code des classes de WPF, afin de trouver des champs privés qui permettraient de récupérer le DataContext de l'élément racine. J'accède ensuite à ces champs par réflexion.  Voici le code :  
```csharp
using System;
using System.Reflection;
using System.Windows;
using System.Windows.Input;
using System.Windows.Markup;

namespace MVVMLib.Input
{
    [MarkupExtensionReturnType(typeof(ICommand))]
    public class CommandBindingExtension : MarkupExtension
    {
        public CommandBindingExtension()
        {
        }

        public CommandBindingExtension(string commandName)
        {
            this.CommandName = commandName;
        }

        [ConstructorArgument("commandName")]
        public string CommandName { get; set; }

        private object targetObject;
        private object targetProperty;

        public override object ProvideValue(IServiceProvider serviceProvider)
        {
            IProvideValueTarget provideValueTarget = serviceProvider.GetService(typeof(IProvideValueTarget)) as IProvideValueTarget;
            if (provideValueTarget != null)
            {
                targetObject = provideValueTarget.TargetObject;
                targetProperty = provideValueTarget.TargetProperty;
            }

            if (!string.IsNullOrEmpty(CommandName))
            {
                // The serviceProvider is actually a ProvideValueServiceProvider, which has a private field "_context" of type ParserContext
                ParserContext parserContext = GetPrivateFieldValue<ParserContext>(serviceProvider, "_context");
                if (parserContext != null)
                {
                    // A ParserContext has a private field "_rootElement", which returns the root element of the XAML file
                    FrameworkElement rootElement = GetPrivateFieldValue<FrameworkElement>(parserContext, "_rootElement");
                    if (rootElement != null)
                    {
                        // Now we can retrieve the DataContext
                        object dataContext = rootElement.DataContext;

                        // The DataContext may not be set yet when the FrameworkElement is first created, and it may change afterwards,
                        // so we handle the DataContextChanged event to update the Command when needed
                        if (!dataContextChangeHandlerSet)
                        {
                            rootElement.DataContextChanged += new DependencyPropertyChangedEventHandler(rootElement_DataContextChanged);
                            dataContextChangeHandlerSet = true;
                        }

                        if (dataContext != null)
                        {
                            ICommand command = GetCommand(dataContext, CommandName);
                            if (command != null)
                                return command;
                        }
                    }
                }
            }

            // The Command property of an InputBinding cannot be null, so we return a dummy extension instead
            return DummyCommand.Instance;
        }

        private ICommand GetCommand(object dataContext, string commandName)
        {
            PropertyInfo prop = dataContext.GetType().GetProperty(commandName);
            if (prop != null)
            {
                ICommand command = prop.GetValue(dataContext, null) as ICommand;
                if (command != null)
                    return command;
            }
            return null;
        }

        private void AssignCommand(ICommand command)
        {
            if (targetObject != null && targetProperty != null)
            {
                if (targetProperty is DependencyProperty)
                {
                    DependencyObject depObj = targetObject as DependencyObject;
                    DependencyProperty depProp = targetProperty as DependencyProperty;
                    depObj.SetValue(depProp, command);
                }
                else
                {
                    PropertyInfo prop = targetProperty as PropertyInfo;
                    prop.SetValue(targetObject, command, null);
                }
            }
        }

        private bool dataContextChangeHandlerSet = false;
        private void rootElement_DataContextChanged(object sender, DependencyPropertyChangedEventArgs e)
        {
            FrameworkElement rootElement = sender as FrameworkElement;
            if (rootElement != null)
            {
                object dataContext = rootElement.DataContext;
                if (dataContext != null)
                {
                    ICommand command = GetCommand(dataContext, CommandName);
                    if (command != null)
                    {
                        AssignCommand(command);
                    }
                }
            }
        }

        private T GetPrivateFieldValue<T>(object target, string fieldName)
        {
            FieldInfo field = target.GetType().GetField(fieldName, BindingFlags.Instance | BindingFlags.NonPublic);
            if (field != null)
            {
                return (T)field.GetValue(target);
            }
            return default(T);
        }

        // A dummy command that does nothing...
        private class DummyCommand : ICommand
        {

            #region Singleton pattern

            private DummyCommand()
            {
            }

            private static DummyCommand _instance = null;
            public static DummyCommand Instance
            {
                get
                {
                    if (_instance == null)
                    {
                        _instance = new DummyCommand();
                    }
                    return _instance;
                }
            }

            #endregion

            #region ICommand Members

            public bool CanExecute(object parameter)
            {
                return false;
            }

            public event EventHandler CanExecuteChanged;

            public void Execute(object parameter)
            {
            }

            #endregion
        }
    }
```
  Cette solution a cependant une limitation : elle ne fonctionne que pour le DataContext de la racine du XAML. On ne peut donc pas l'utiliser, par exemple, pour définir des InputBindings sur un contrôle dont on redéfinit aussi le DataContext, car la markup extension renverra le DataContext de l'élément racine.

