---
layout: post
title: '[WinRT] Sélectionner un élément de liste lors d’un appui long'
date: 2013-11-22T00:00:00.0000000
url: /2013/11/22/winrt-selectionner-un-element-de-liste-lors-d-un-appui-long/
tags:
  - appui long
  - geste
  - GridView
  - hold
  - ListView
  - selection
  - winrt
categories:
  - WinRT
---


Comme vous le savez probablement, la méthode standard pour sélectionner ou déselectionner un élément dans un contrôle de liste WinRT est de le faire glisser légèrement vers le haut ou vers le bas. Même si j’aime bien ce geste, il n’est pas très intuitif pour les utilisateurs qui ne sont pas habitués à Modern UI. Et ça devient encore plus déroutant, car ma déclaration précédente n’est pas tout à fait exacte : en fait, il faut faire glisser l’élément *perpendiculairement à la direction de défilement* de la liste. Dans une `GridView`, qui défile horizontalement (par défaut), c’est vers le haut ou vers le bas ; mais pour une `ListView`, qui défile verticalement, il faut faire glisser l’élément vers la gauche ou vers la droite. Si une application utilise les deux types de liste, ça devient vraiment *très* déroutant pour l’utilisateur.

Bien sûr, dans le style par défaut, il y a une indication visuelle (une discrète animation “glissement vers le bas” avec une coche grise) quand l’utilisateur appuie longuement sur un élément, mais ce n’est pas toujours suffisant pour que tout le monde comprenne. Beaucoup de gens (par exemple les utilisateurs d’Android) on l’habitude de sélectionner les éléments par un appui long (appelé “Hold” dans la terminologie Modern UI). Donc, pour rendre votre application facilement utilisable par le plus grand nombre d’utilisateurs, il peut être intéressant de permettre la sélection par un appui long.

Un moyen simple de le faire est de créer une propriété attachée qui, quand on la met à `true`, s’abonne à l’évènement `Holding` de l’élément, et change la valeur de la propriété `IsSelected` quand l’évènement se produit. Voilà une implémentation possible :



```
using Windows.UI.Input;
using Windows.UI.Xaml;
using Windows.UI.Xaml.Controls.Primitives;
using Windows.UI.Xaml.Input;

namespace TestSelectOnHold
{
    public static class SelectorItemEx
    {
        public static bool GetToggleSelectedOnHold(SelectorItem item)
        {
            return (bool)item.GetValue(ToggleSelectedOnHoldProperty);
        }

        public static void SetToggleSelectedOnHold(SelectorItem item, bool value)
        {
            item.SetValue(ToggleSelectedOnHoldProperty, value);
        }

        public static readonly DependencyProperty ToggleSelectedOnHoldProperty =
            DependencyProperty.RegisterAttached(
              "ToggleSelectedOnHold",
              typeof(bool),
              typeof(SelectorItemEx),
              new PropertyMetadata(
                false,
                ToggleSelectedOnHoldChanged));

        private static void ToggleSelectedOnHoldChanged(DependencyObject o, DependencyPropertyChangedEventArgs e)
        {
            var item = o as SelectorItem;
            if (item == null)
                return;

            var oldValue = (bool)e.OldValue;
            var newValue = (bool)e.NewValue;

            if (oldValue && !newValue)
            {
                item.Holding -= Item_Holding;
            }
            else if (newValue && !oldValue)
            {
                item.Holding += Item_Holding;
            }
        }

        private static void Item_Holding(object sender, HoldingRoutedEventArgs e)
        {
            var item = sender as SelectorItem;
            if (item == null)
                return;

            if (e.HoldingState == HoldingState.Started)
                item.IsSelected = !item.IsSelected;
        }
    }
}
```

Vous pouvez ensuite définir cette propriété dans l’`ItemContainerStyle` du contrôle de liste :

```
<GridView.ItemContainerStyle>
    <Style TargetType="GridViewItem">
        ...
        <Setter Property="local:SelectorItemEx.ToggleSelectedOnHold" Value="False" />
    </Style>
</GridView.ItemContainerStyle>
```

Et c’est tout : l’utilisateur peut maintenant sélectionner les éléments par un appui long. Le geste standard fonctionne toujours, bien sûr, donc les utilisateurs qui le connaissent peuvent toujours l’utiliser.

Notez que cette fonctionnalité aurait aussi pu être implémentée comme un `Behavior` à part entière. Il y a deux raisons pour lesquelles je n’ai pas choisi cette approche :

- Les Behaviors ne sont pas supportés nativement dans WinRT (bien qu’on puisse les ajouter avec un [package Nuget](http://www.nuget.org/packages/Windows.UI.Interactivity/))
- Les Behaviors ne fonctionnent pas très bien avec les styles, parce que `Interaction.Behaviors` est une collection, et on ne peut pas ajouter des éléments à une collection dans un style. Une solution possible pour contourner le problème serait de créer une propriété attachée `IsEnabled`, qui ajouterait le behavior à la collection quand on la met à `true`, mais on se retrouverait finalement avec une solution quasiment identique à celle décrite plus haut, en plus complexe…


