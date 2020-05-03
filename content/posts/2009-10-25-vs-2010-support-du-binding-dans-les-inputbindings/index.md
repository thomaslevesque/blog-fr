---
layout: post
title: '[VS 2010] Support du binding dans les InputBindings'
date: 2009-10-25T00:00:00.0000000
url: /2009/10/25/vs-2010-support-du-binding-dans-les-inputbindings/
tags:
  - .NET 4.0
  - binding
  - InputBinding
  - MVVM
  - Visual Studio
  - Visual Studio 2010
  - WPF
  - wpf 4.0
  - XAML
categories:
  - WPF
---

**LA fonctionnalité qui manquait à WPF !**  La beta 2 de Visual Studio 2010 est là depuis quelques jours, et apporte à WPF une nouveauté que j'attendais depuis longtemps : le support du binding dans les `InputBindings`.  Pour rappel, le problème de la version précédente était que la propriété `Command` de la classe `InputBinding` n'était pas une `DependencyProperty`, on ne pouvait donc pas la définir par un binding. D'ailleurs, les `InputBindings` n'héritaient pas du `DataContext`, ce qui compliquait beaucoup les implémentations alternatives de cette fonctionnalité...  Jusqu'ici, pour lier la commande d'un `KeyBinding` ou `MouseBinding` à une propriété du `DataContext`, il fallait donc passer par des détours pas forcément très élégants... J'avais fini par trouver une solution acceptable, détaillée dans [ce post](http://www.thomaslevesque.fr/2009/03/17/wpf-utiliser-les-inputbindings-avec-le-pattern-mvvm/), mais qui me laissait assez insatisfait (utilisation de la réflexion sur des membres privés, pas mal de limitations...).  J'ai découvert plus récemment le [MVVM toolkit](http://www.codeplex.com/wpf/Release/ProjectReleases.aspx?ReleaseId=14962), qui propose une approche un peu plus propre : une classe `CommandReference`, héritée de `Freezable`, qui permet de mettre dans les ressources de la page ou du contrôle une référence à la commande, qu'on peut ensuite utiliser avec `StaticResource`. Plus propre, mais ça restait peu intuitif...  WPF 4.0 résout le problème une bonne fois pour toutes : la classe `InputBinding` hérite maintenant de `Freezable`, ce qui lui permet d'hériter du `DataContext` parent. De plus les propriétés `Command`, `CommandParameter` et `CommandTarget` sont maintenant des `DependencyProperty`. On peut donc laisser tomber tous les bricolages qu'il fallait utiliser jusqu'à maintenant, et aller droit au but :  
```xml
    <Window.InputBindings>
        <KeyBinding Key="F5" Command="{Binding RefreshCommand}" />
    </Window.InputBindings>
```
  Voilà qui devrait faciliter un peu le développement d'applications MVVM !  **Help 3**  Je change un peu de sujet pour vous parler du nouveau système de documentation offline de Visual Studio 2010, appelé "Help 3". Il se présente comme une application web installée localement, les pages s'affichent donc dans le navigateur. Globalement, c'est beaucoup mieux... largement plus léger et plus réactif que le vieux Document Explorer livré avec les versions précédentes de Visual Studio.  En revanche, une fonction qui m'était absolument indispensable a disparu : l'index ! Il ne reste que la vue en arborescence et la recherche. Plus de trace de l'index que j'utilisais en permanence pour accéder directement à une classe ou un membre, sans forcément connaître son namespace. En plus, les résultats de la recherche ne montre pas clairement le namespace : par exemple, si vous tapez "button class" dans la recherche, pas moyen de voir la différence entre `System.Windows.Forms.Button`, `System.Windows.Controls.Button` et `System.Web.UI.WebControls` ! Avant, le volet "Résultats de l'index" affichait cette information clairement.  Mon sentiment sur ce nouveau système d'aide est donc un peu mitigé... il va falloir que je change ma façon d'utiliser la doc. Mais à part ce détail agaçant, c'est objectivement un gros progrès !

