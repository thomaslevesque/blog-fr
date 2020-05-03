---
layout: post
title: '[WPF] Déclarer des raccourcis clavier globaux en XAML avec NHotkey'
date: 2014-02-11T00:00:00.0000000
url: /2014/02/11/wpf-declare-global-hotkeys-in-xaml-with-nhotkey/
tags:
  - global
  - hotkey
  - raccourci clavier
  - Windows Forms
  - WPF
  - XAML
categories:
  - Librairies
---


Un besoin fréquent pour les applications de bureau est de gérer des raccourcis claviers globaux, pour pouvoir réagir aux raccourcis même quand l’application n’a pas le focus. Malheureusement, il n’y aucune fonctionnalité intégrée dans le .NET Framework pour gérer ça.

Bien sûr, le problème n’est pas nouveau, et il y a un certain nombre de librairies open-source qui se proposent d’y remédier (par exemple [VirtualInput](https://github.com/SaqibS/VirtualInput)). La plupart d’entre elles sont basées sur des hooks système globaux, ce qui leur permet d’intercepter *toutes* les frappes de touche, même celles qui ne vous intéressent pas. J’ai déjà utilisé certaines de ces librairies, mais je n’en suis pas vraiment satisfait :

- elles sont souvent liées à un framework IHM spécifique (généralement Windows Forms), ce qui les rend peu pratiques à utiliser avec un autre framework (comme WPF)
- je n’aime pas trop l’approche consistant à intercepter toutes les frappes de touche. Cela a généralement pour conséquence d’écrire un grosse méthode avec un paquet de `if/else if` pour décider quoi faire en fonction de la combinaison de touches qui a été pressée


Une meilleure option, à mon sens, est d’écouter uniquement les touches qui vous intéressent, et de spécifier de façon déclarative l’action à effectuer pour chacune d’entre elles. L’approche utilisée en WPF pour les `KeyBindings` est assez élégante :

```
<Window.InputBindings>
    <KeyBinding Gesture="Ctrl+Alt+Add" Command="{Binding IncrementCommand}" />
    <KeyBinding Gesture="Ctrl+Alt+Subtract" Command="{Binding DecrementCommand}" />
</Window.InputBindings>
```

Mais bien sûr, les `KeyBindings` ne fonctionnent pas de façon globale, ils nécessitent que votre application ait le focus… Et si on pouvait changer ça ?

[NHotkey](https://github.com/thomaslevesque/NHotkey) est une librairie très simple pour gérer les raccourcis clavier, qui permet entre autres d’avoir des `KeyBindings` globaux. Il suffit de mettre à true une propriété attachée sur le `KeyBinding` :

```
<Window.InputBindings>
    <KeyBinding Gesture="Ctrl+Alt+Add" Command="{Binding IncrementCommand}"
                HotkeyManager.RegisterGlobalHotkey="True" />
    <KeyBinding Gesture="Ctrl+Alt+Subtract" Command="{Binding DecrementCommand}"
                HotkeyManager.RegisterGlobalHotkey="True" />
</Window.InputBindings>
```

Et c’est tout ; les commandes définies sur les `KeyBindings` seront maintenant invoquées même si votre application n’a pas le focus !

Vous pouvez aussi utiliser NHotkey depuis le code :

```
HotkeyManager.Current.AddOrReplace("Increment", Key.Add, ModifierKeys.Control | ModifierKeys.Alt, OnIncrement);
HotkeyManager.Current.AddOrReplace("Decrement", Key.Subtract, ModifierKeys.Control | ModifierKeys.Alt, OnDecrement);
```

La librairie tire partie de la fonction `RegisterHotkey`. Parce qu’elle supporte également Windows Forms, elle est constituée de 3 parties distinctes, afin de ne pas avoir à référencer l’assembly Windows Forms depuis une appli WPF, ou vice versa :

- La librarie “core”, qui gère l’enregistrement des hotkeys proprement dit, indépendamment d’un framework IHM spécifique. Cette librairie n’est pas directement utilisable, mais elle est utilisée par les deux autres.
- L’API spécifique à WinForms, qui utilise l’énumération `Keys` de `System.Windows.Forms`.
- L’API spécifique à WPF, qui utilise les énumérations `Key` et `ModifierKeys` de `System.Windows.Input`, et supporte les `KeyBindings` globaux en XAML.


Si vous installez la librairie à l’aide de Nuget, ajoutez l’un ou l’autre des packages [NHotkey.Wpf](http://www.nuget.org/packages/NHotkey.Wpf/) ou [NHotkey.WindowsForms](http://www.nuget.org/packages/NHotkey.WindowsForms/) ; le package “core” sera automatiquement ajouté.

