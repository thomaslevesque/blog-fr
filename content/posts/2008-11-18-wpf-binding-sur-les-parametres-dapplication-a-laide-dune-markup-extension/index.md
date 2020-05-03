---
layout: post
title: '[WPF] Binding sur les paramètres d’application à l’aide d’une Markup extension'
date: 2008-11-18T00:00:00.0000000
url: /2008/11/18/wpf-binding-sur-les-parametres-dapplication-a-laide-dune-markup-extension/
tags:
  - binding
  - markup extension
  - paramètres
  - settings
  - WPF
  - XAML
categories:
  - Code sample
  - WPF
---

Voilà, c’est fait, j'ai créé mon blog sur .NET... j'ai mis le temps, mais j'ai fini par y venir ;-)  Je me présente rapidement : Thomas Levesque, 27 ans, ingénieur de formation. Je suis passionné depuis toujours par l'informatique, et plus particulièrement par la technologie .NET, que je suis de très près depuis ses débuts. Comme je suis du genre curieux, je passe pas mal de temps à fouiner dans les docs MSDN et sur le net pour m'auto-former sur les dernières nouveautés du framework. Aujourd'hui, je travaille comme développeur C# pour une PME en région parisienne.  Mais assez parlé de moi, venons-en au sujet qui nous intéresse : le binding sur les paramètres d'application en WPF.  Pour l'utilisateur d'une application, il est plus confortable de ne pas avoir à redéfinir sans arrêt ses préférences (dimensions de la fenêtre, activation de telle ou telle option...) : d'où l'utilité des paramètres d'application, introduits dans la version 2.0 du framework. Pour le développeur, c'est un peu pénible... même avec la classe Settings définie par Visual Studio, il reste encore pas mal de code à écrire pour lire et appliquer les paramètres au démarrage de l'appli, puis les enregistrer avant de quitter.  Dans Windows Forms, on pouvait définir des bindings entre les propriétés des contrôles et les paramètres d'application, mais ça restait peu pratique, et finalement assez peu utilisé (du moins c'est l'impression que j'en ai...).  Avec WPF, ça devient beaucoup plus intéressant... bien qu'il n'y ait pas de documentation "officielle" sur cette pratique, il est tout à fait possible de définir dans le code XAML des bindings sur les paramètres d'application. Par exemple, pour binder les dimensions de la fenêtre sur des paramètres, la méthode qui est présentée sur de nombreux blogs est la suivante :  
```
<Window x:Class="WpfApplication1.Window1"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:p="clr-namespace:WpfApplication1.Properties"
        Title="Window1"
        Height="{Binding Source={x:Static p:Settings.Default}, Path=Height, Mode=TwoWay}"
        Width="{Binding Source={x:Static p:Settings.Default}, Path=Width, Mode=TwoWay}"
        Left="{Binding Source={x:Static p:Settings.Default}, Path=Left, Mode=TwoWay}"
        Top="{Binding Source={x:Static p:Settings.Default}, Path=Top, Mode=TwoWay}">
```
  (Dans cet exemple, Height, Width, Top et Left sont des paramètres de l’application)  Cette méthode a l’avantage de fonctionner, mais franchement, est-ce que vous vous voyez écrire ça des dizaines de fois, pour chaque paramètre de l’application ? C’est long à écrire, peu intuitif, répétitif, et ça rend le code globalement peu lisible…  Loin de moi l’idée de dénigrer ceux qui ont eu cette idée, bien sûr… mais il est très facile d’améliorer cette méthode, en créant notre propre « markup extension ». Nous allons donc définir une classe qui va hériter de Binding, et servir spécifiquement à lier des propriétés à des paramètres de l’application.  Une « markup extension » (qu’on pourrait traduire approximativement par « balise d’extension ») est un objet qu’on peut utiliser dans le code XAML pour récupérer une valeur. On en utilise en permanence en WPF : Binding, StaticResource, DynamicResource sont des exemples de markup extensions.  On peut facilement définir sa propre markup extension en créant une classe dérivée de MarkupExtension, qui doit implémenter une méthode ProvideValue. En l’occurrence, l’essentiel de ce qu’on veut faire est déjà implémenté dans la classe Binding (qui hérite indirectement de MarkupExtension). Nous allons donc hériter directement de Binding, et simplement initialiser les propriétés qui nous intéressent pour se binder sur les paramètres :  
```csharp
using System.Windows.Data;

namespace WpfApplication1
{
    public class SettingBindingExtension : Binding
    {
        public SettingBindingExtension()
        {
            Initialize();
        }

        public SettingBindingExtension(string path)
            :base(path)
        {
            Initialize();
        }

        private void Initialize()
        {
            this.Source = WpfApplication1.Properties.Settings.Default;
            this.Mode = BindingMode.TwoWay;
        }
    }
}
```
  Notez le suffixe « Extension » à la fin du nom de la classe : par convention, la plupart des markup extensions ont ce suffixe (Binding est une exception qui confirme la règle…). Ce suffixe pourra être omis quand on utilisera la classe en XAML (un peu comme les attributs, dont on omet le suffixe « Attribute » ).  Dans cette classe, on a défini 2 constructeurs, qui correspondent à ceux de Binding. On ne redéfinit pas la méthode ProvideValue, car celle de la classe Binding nous convient parfaitement (et d’ailleurs elle est marquée comme finale (sealed), si bien qu’on ne pourrait pas la redéfinir…). Le code « intéressant », si j’ose dire, est dans la méthode Initialize. On y définit la propriété Source comme étant les paramètres de notre appli, de façon à ce que le Path indiqué pour le binding renvoie le paramètre qui nous intéresse, et Mode = TwoWay, de façon à ce que les paramètres soient automatiquement mis à jour à partir de l’interface. L’intérêt étant de ne pas avoir à redéfinir ces propriétés à chaque fois…  Pour utiliser cette classe, c’est tout simple ! Reprenons l’exemple précédent, en remplaçant les Binding par notre extension SettingBinding :  
```xml
<Window x:Class="WpfApplication1.Window1"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:my="clr-namespace:WpfApplication1"
        Title="Window1"
        Height="{my:SettingBinding Height}"
        Width="{my:SettingBinding Width}"
        Left="{my:SettingBinding Left}"
        Top="{my:SettingBinding Top}">
```
  C’est tout de même plus lisible et plus facile à utiliser…  Ah, et bien sûr, pour que ça marche, on n’oublie pas d’enregistrer les paramètres dans l’évènement Exit de l’application…  
```csharp
        private void Application_Exit(object sender, ExitEventArgs e)
        {
            WpfApplication1.Properties.Settings.Default.Save();
        }
```
  Et voilà, les dimensions de la fenêtre seront enregistrées et restaurées à chaque lancement de l’application, sans qu’on ait rien de plus à coder !  [Télécharger les sources](http://www.thomaslevesque.fr/files/2012/06/SettingBindingSample.zip)**Mise à jour :** Si vous voulez en savoir plus sur les markup extensions, je vous invite à lire le tutoriel que j'ai écrit suite à ce billet : [Les markup extensions en WPF](http://tlevesque.developpez.com/dotnet/wpf-markup-extensions/)

