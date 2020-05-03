---
layout: post
title: '[WPF] Comment faire un binding dans les cas où on n’hérite pas du DataContext'
date: 2011-03-21T00:00:00.0000000
url: /2011/03/21/wpf-comment-faire-un-binding-dans-les-cas-o-on-nhrite-pas-du-datacontext/
tags:
  - binding
  - datacontext
  - freezable
  - WPF
categories:
  - Astuces
  - WPF
---

La propriété `DataContext` de WPF est extrêmement pratique, car elle est automatiquement héritée par tous les enfants de l’élément où elle est définie ; il n’est donc pas nécessaire de la redéfinir pour chaque élément qu’on veut lier aux données. Cependant, il arrive que le `DataContext` ne soit pas accessible pour certains éléments : c’est le cas des éléments qui n’appartiennent pas à l’arbre visuel ni à l’arbre logique. Il devient alors très difficile de définir une propriété ce ces éléments par un binding...  Prenons un exemple simple : on veut afficher une liste de produits dans un `DataGrid`. Dans la grille, on veut pouvoir afficher où masquer la colonne du prix, en fonction d’une propriété `ShowPrice` exposée par le ViewModel. L’approche évidente consiste à binder la propriété `Visibility` de la colonne à la propriété `ShowPrice` :  
```xml
<DataGridTextColumn Header="Price" Binding="{Binding Price}" IsReadOnly="False"
                    Visibility="{Binding ShowPrice,
                        Converter={StaticResource visibilityConverter}}"/>
```
  Malheureusement, changer la valeur de la propriété `ShowPrice` n'a aucun effet, et la colonne reste toujours affichée... pourquoi ? Si on regarde la fenêtre de sortie de Visual Studio, on remarque la ligne suivante :  

> System.Windows.Data Error: 2 : Cannot find governing FrameworkElement or FrameworkContentElement for target element. BindingExpression:Path=ShowPrice; DataItem=null; target element is 'DataGridTextColumn' (HashCode=32685253); target property is 'Visibility' (type 'Visibility')

  Derrière cet obscur charabia se cache une explication toute simple : WPF ne sait pas quel `FrameworkElement` utiliser pour récupérer le `DataContext`, car la colonne n'appartient pas à l'arbre visuel ni à l'arbre logique du `DataGrid`.  On peut toujours essayer de "triturer" le binding pour obtenir le résultat voulu, par exemple en essayant de binder par rapport au `DataGrid` lui-même :  
```xml
<DataGridTextColumn Header="Price" Binding="{Binding Price}" IsReadOnly="False"
                    Visibility="{Binding DataContext.ShowPrice,
                        Converter={StaticResource visibilityConverter},
                        RelativeSource={RelativeSource FindAncestor, AncestorType=DataGrid}}"/>
```
  Ou encore, en ajoutant une `CheckBox` bindée sur `ShowPrice` et en essayant de binder la visibilité de la colonne sur la propriété `IsChecked`, en spécifiant le nom de l'élément :  
```xml
<DataGridTextColumn Header="Price" Binding="{Binding Price}" IsReadOnly="False"
                    Visibility="{Binding IsChecked,
                        Converter={StaticResource visibilityConverter},
                        ElementName=chkShowPrice}"/>
```
  Mais rien à faire, on obtient toujours le même résultat...  A ce stade, il semble que la seule approche qui pourrait marcher est de passer par le code-behind, ce qu'on préfère généralement éviter quand on suit le pattern MVVM... mais ce serait dommage d'abandonner aussi vite ;)  La solution est en fait assez simple, et se base sur la classe [`Freezable`](http://msdn.microsoft.com/fr-fr/library/system.windows.freezable.aspx). La vocation première de cette classe est de définir des objets qui ont un état modifiable et un état non modifiable. Mais en l'occurrence, la caractéristique qui nous intéresse est qu'un objet qui hérite de `Freezable` peut hériter du `DataContext`, bien qu'il ne s'agisse pas d'un élément visuel. Je ne connais pas le mécanisme exact qui permet d'obtenir ce comportement, mais toujours est-il que cela va nous permettre d'arriver au résultat voulu...  L'idée est de créer une classe, que j'ai appelée `BindingProxy`, qui hérite de `Freezable` et dans laquelle on va déclarer une dependency property `Data` :  
```csharp
    public class BindingProxy : Freezable
    {
        #region Overrides of Freezable

        protected override Freezable CreateInstanceCore()
        {
            return new BindingProxy();
        }

        #endregion

        public object Data
        {
            get { return (object)GetValue(DataProperty); }
            set { SetValue(DataProperty, value); }
        }

        // Using a DependencyProperty as the backing store for Data.  This enables animation, styling, binding, etc...
        public static readonly DependencyProperty DataProperty =
            DependencyProperty.Register("Data", typeof(object), typeof(BindingProxy), new UIPropertyMetadata(null));
    }
```
  On va ensuite déclarer une instance de cette classe dans les ressources du `DataGrid`, et binder la propriété `Data` sur le `DataContext` courant :  
```xml
<DataGrid.Resources>
    <local:BindingProxy x:Key="proxy" Data="{Binding}" />
</DataGrid.Resources>
```
  Il suffit ensuite de spécifier que la source de notre binding est cet objet `BindingProxy`, facilement accessible puisqu'il est déclaré comme ressource :  
```xml
<DataGridTextColumn Header="Price" Binding="{Binding Price}" IsReadOnly="False"
                    Visibility="{Binding Data.ShowPrice,
                        Converter={StaticResource visibilityConverter},
                        Source={StaticResource proxy}}"/>
```
  Remarquez qu'on a aussi préfixé le chemin du binding par "Data", puisque le chemin est maintenant relatif à l'objet `BindingProxy`.  Le binding fonctionne maintenant comme prévu, moyennant une solution relativement simple à mettre en oeuvre...

