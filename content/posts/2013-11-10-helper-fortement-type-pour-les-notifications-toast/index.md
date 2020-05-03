---
layout: post
title: Helper fortement typé pour les notifications toast
date: 2013-11-10T00:00:00.0000000
url: /2013/11/10/helper-fortement-type-pour-les-notifications-toast/
tags:
  - notification
  - toast
  - windows store
  - winrt
categories:
  - WinRT
---


Windows 8 fournit une API pour afficher des notifications toast. Malheureusement, elle est très peu pratique à utiliser : pour définir le contenu d’une notification, il faut utiliser un modèle prédéfini qui est fourni sous la forme d’un `XmlDocument`, et fixer la valeur de chaque champ dans le XML. Il n’y a rien dans l’API pour indiquer quels champs sont définis dans le modèle et à quoi ils correspondent, il faut consulter le [catalogue de modèles de toast](http://msdn.microsoft.com/fr-fr/library/windows/apps/hh761494.aspx) dans la documentation. Il serait beaucoup plus pratique d’avoir une API fortement typée…

J’ai donc créé un simple wrapper autour de l’API des toasts. On peut l’utiliser comme ceci :

```
var content = new ToastContent.ImageAndText02
{
    Image = "ms-appx:///Images/dotnet.png",
    Title = "Hello world!",
    Text = "Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
};
var notifier = ToastNotificationManager.CreateToastNotifier();
notifier.Show(content.CreateNotification());
```

Notez que j’ai gardé les noms d’origine du catalogue de modèles, parce que des noms suffisamment descriptifs auraient été trop longs. J’ai inclus des commentaires XML de documentation pour faciliter le choix du modèle.

Si vous voulez plus de flexibilité qu’un modèle fortement typé ne peut en offrir, mais que vous ne voulez pas manipuler le XML du modèle, vous pouvez utiliser la classe `ToastContent` directement :

```
var content = new ToastContent(ToastTemplateType.ToastImageAndText02);
content.SetImage(1, "ms-appx:///Images/dotnet.png");
content.SetText(1, "Hello world!");
content.SetText(2, "Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.");
var notifier = ToastNotificationManager.CreateToastNotifier();
notifier.Show(content.CreateNotification());
```


Le code est [disponible sur Github](https://github.com/thomaslevesque/ToastHelper), avec une application de démo. Un [package NuGet](https://www.nuget.org/packages/ToastHelper/) est également disponible.

Un point intéressant est la façon dont j’ai créé les classes de modèle : j’aurais pu le faire à la main, mais cela aurait été assez fastidieux. J’ai donc extrait les modèles de toast dans un fichier XML, je l’ai enrichi de quelques informations supplémentaires (noms des propriétés, description pour les commentaires de documentation), et j’ai créé un template T4 pour générer automatiquement les classes à partir du fichier XML.

