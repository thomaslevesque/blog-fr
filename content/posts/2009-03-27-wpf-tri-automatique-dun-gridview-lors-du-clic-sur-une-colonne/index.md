---
layout: post
title: '[WPF] Tri automatique d''un GridView lors du clic sur une colonne'
date: 2009-03-27T00:00:00.0000000
url: /2009/03/27/wpf-tri-automatique-dun-gridview-lors-du-clic-sur-une-colonne/
tags:
  - GridView
  - MVVM
  - propriété attachée
  - tri
  - WPF
categories:
  - Code sample
  - WPF
---

Il est assez simple, en WPF, de présenter des données sous forme de grille, grâce à la classe `GridView`. Pour le tri, en revanche, ça se complique... Avec le `DataGridView` de Windows Forms, c'était "automagique" : quand l'utilisateur cliquait sur un en-tête de colonne, le tri se faisait automatiquement sur cette colonne. En WPF, par contre, il faut un peu mettre les mains dans le cambouis... La méthode préconisée par Microsoft pour trier un `GridView` lors du clic sur une colonne est décrite dans [cet article](http://msdn.microsoft.com/fr-fr/library/ms745786.aspx) ; elle est basée sur l'évènement `Click` du `GridViewColumnHeader`. A mon sens, cette méthode présente deux gros inconvénients : 
- Le tri doit être réalisé dans le code-behind, ce qu'on préfère souvent éviter si on s'appuie sur un design pattern comme MVVM. De plus, cela rend le code moins facilement réutilisable
- Cette méthode suppose que le texte de l'en-tête de la colonne correspond au nom de la propriété sur laquelle on veut trier. Ce qui, bien sûr, est loin d'être toujours le cas... On pourrait se baser sur le `DisplayMemberBinding` de la colonne, mais il n'est pas forcément défini (par exemple si on définit un `CellTemplate` à la place).

  Après avoir longuement tâtonné pour trouver une approche souple et élégante, j'ai fini par réaliser une classe `GridViewSort` qui permet de trier automatiquement un `GridView` selon des propriétés attachées définies en XAML.  On utilise cette classe de la façon suivante :  
```xml
                <ListView ItemsSource="{Binding Persons}"
                      IsSynchronizedWithCurrentItem="True"
                      util:GridViewSort.AutoSort="True">
                    <ListView.View>
                        <GridView>
                            <GridView.Columns>
                                <GridViewColumn Header="Nom"
                                                DisplayMemberBinding="{Binding Name}"
                                                util:GridViewSort.PropertyName="Name"/>
                                <GridViewColumn Header="Prénom"
                                                DisplayMemberBinding="{Binding FirstName}"
                                                util:GridViewSort.PropertyName="FirstName"/>
                                <GridViewColumn Header="Date de naissance"
                                                DisplayMemberBinding="{Binding DateOfBirth}"
                                                util:GridViewSort.PropertyName="DateOfBirth"/>
                            </GridView.Columns>
                        </GridView>
                    </ListView.View>
                </ListView>
```
  La propriété `GridViewSort.AutoSort` active le tri automatique pour la `ListView`. La propriété `GridViewSort.PropertyName`, définie sur chaque colonne, indique sur quelle propriété le tri doit être effectué. Il n'y a aucun code supplémentaire à écrire. Le clic sur un en-tête de colonne déclenche le tri sur cette colonne ; si la `ListView` est déjà triée sur cette colonne, l'ordre de tri est inversé.  Pour le cas où on souhaiterait gérer manuellement le tri, j'ai aussi créé une propriété attachée `GridViewSort.Command`. Par exemple, dans le cadre de l'utilisation du pattern MVVM, on peut binder cette propriété sur une commande déclarée dans le ViewModel :  
```xml
                <ListView ItemsSource="{Binding Persons}"
                      IsSynchronizedWithCurrentItem="True"
                      util:GridViewSort.Command="{Binding SortCommand}">
                ...
```
  La commande de tri reçoit en paramètre le nom de la propriété sur laquelle on veut trier.  Note : si les propriétés `Command` et `AutoSort` sont définies toutes les deux, c'est `Command` qui est prioritaire ; `AutoSort` est ignorée.  Voici le code complet de la classe `GridViewSort` :  
```csharp
using System.ComponentModel;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Input;
using System.Windows.Media;

namespace Wpf.Util
{
    public class GridViewSort
    {
        #region Attached properties

        public static ICommand GetCommand(DependencyObject obj)
        {
            return (ICommand)obj.GetValue(CommandProperty);
        }

        public static void SetCommand(DependencyObject obj, ICommand value)
        {
            obj.SetValue(CommandProperty, value);
        }

        // Using a DependencyProperty as the backing store for Command.  This enables animation, styling, binding, etc...
        public static readonly DependencyProperty CommandProperty =
            DependencyProperty.RegisterAttached(
                "Command",
                typeof(ICommand),
                typeof(GridViewSort),
                new UIPropertyMetadata(
                    null,
                    (o, e) =>
                    {
                        ItemsControl listView = o as ItemsControl;
                        if (listView != null)
                        {
                            if (!GetAutoSort(listView)) // Don't change click handler if AutoSort enabled
                            {
                                if (e.OldValue != null && e.NewValue == null)
                                {
                                    listView.RemoveHandler(GridViewColumnHeader.ClickEvent, new RoutedEventHandler(ColumnHeader_Click));
                                }
                                if (e.OldValue == null && e.NewValue != null)
                                {
                                    listView.AddHandler(GridViewColumnHeader.ClickEvent, new RoutedEventHandler(ColumnHeader_Click));
                                }
                            }
                        }
                    }
                )
            );

        public static bool GetAutoSort(DependencyObject obj)
        {
            return (bool)obj.GetValue(AutoSortProperty);
        }

        public static void SetAutoSort(DependencyObject obj, bool value)
        {
            obj.SetValue(AutoSortProperty, value);
        }

        // Using a DependencyProperty as the backing store for AutoSort.  This enables animation, styling, binding, etc...
        public static readonly DependencyProperty AutoSortProperty =
            DependencyProperty.RegisterAttached(
                "AutoSort",
                typeof(bool),
                typeof(GridViewSort),
                new UIPropertyMetadata(
                    false,
                    (o, e) =>
                    {
                        ListView listView = o as ListView;
                        if (listView != null)
                        {
                            if (GetCommand(listView) == null) // Don't change click handler if a command is set
                            {
                                bool oldValue = (bool)e.OldValue;
                                bool newValue = (bool)e.NewValue;
                                if (oldValue && !newValue)
                                {
                                    listView.RemoveHandler(GridViewColumnHeader.ClickEvent, new RoutedEventHandler(ColumnHeader_Click));
                                }
                                if (!oldValue && newValue)
                                {
                                    listView.AddHandler(GridViewColumnHeader.ClickEvent, new RoutedEventHandler(ColumnHeader_Click));
                                }
                            }
                        }
                    }
                )
            );

        public static string GetPropertyName(DependencyObject obj)
        {
            return (string)obj.GetValue(PropertyNameProperty);
        }

        public static void SetPropertyName(DependencyObject obj, string value)
        {
            obj.SetValue(PropertyNameProperty, value);
        }

        // Using a DependencyProperty as the backing store for PropertyName.  This enables animation, styling, binding, etc...
        public static readonly DependencyProperty PropertyNameProperty =
            DependencyProperty.RegisterAttached(
                "PropertyName",
                typeof(string),
                typeof(GridViewSort),
                new UIPropertyMetadata(null)
            );

        #endregion

        #region Column header click event handler

        private static void ColumnHeader_Click(object sender, RoutedEventArgs e)
        {
            GridViewColumnHeader headerClicked = e.OriginalSource as GridViewColumnHeader;
            if (headerClicked != null)
            {
                string propertyName = GetPropertyName(headerClicked.Column);
                if (!string.IsNullOrEmpty(propertyName))
                {
                    ListView listView = GetAncestor<ListView>(headerClicked);
                    if (listView != null)
                    {
                        ICommand command = GetCommand(listView);
                        if (command != null)
                        {
                            if (command.CanExecute(propertyName))
                            {
                                command.Execute(propertyName);
                            }
                        }
                        else if (GetAutoSort(listView))
                        {
                            ApplySort(listView.Items, propertyName);
                        }
                    }
                }
            }
        }

        #endregion

        #region Helper methods

        public static T GetAncestor<T>(DependencyObject reference) where T : DependencyObject
        {
            DependencyObject parent = VisualTreeHelper.GetParent(reference);
            while (!(parent is T))
            {
                parent = VisualTreeHelper.GetParent(parent);
            }
            if (parent != null)
                return (T)parent;
            else
                return null;
        }

        public static void ApplySort(ICollectionView view, string propertyName)
        {
            ListSortDirection direction = ListSortDirection.Ascending;
            if (view.SortDescriptions.Count > 0)
            {
                SortDescription currentSort = view.SortDescriptions[0];
                if (currentSort.PropertyName == propertyName)
                {
                    if (currentSort.Direction == ListSortDirection.Ascending)
                        direction = ListSortDirection.Descending;
                    else
                        direction = ListSortDirection.Ascending;
                }
                view.SortDescriptions.Clear();
            }
            if (!string.IsNullOrEmpty(propertyName))
            {
                view.SortDescriptions.Add(new SortDescription(propertyName, direction));
            }
        }

        #endregion
    }
}
```
  On pourrait bien sûr envisager certaines améliorations, notamment visuelles, comme l'ajout d'une flèche sur la colonne triée (à l'aide d'un `Adorner` par exemple). Mais en attendant, cette classe couvre tout l'aspect fonctionnel du tri, donc n'hésitez pas à l'utiliser !

  **Mise à jour :** l'affichage du symbole de tri est maintenant géré par la classe `GridViewSort`, la nouvelle version est disponible dans [ce billet](/2009/08/04/wpf-tri-automatique-dun-gridview-suite/).

