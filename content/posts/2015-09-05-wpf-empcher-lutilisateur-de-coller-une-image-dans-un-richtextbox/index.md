---
layout: post
title: '[WPF] Empêcher l’utilisateur de coller une image dans un RichTextBox'
date: 2015-09-05T00:00:00.0000000
url: /2015/09/05/wpf-empcher-lutilisateur-de-coller-une-image-dans-un-richtextbox/
tags:
  - C#
  - image
  - presse-papier
  - richtextbox
  - WPF
categories:
  - WPF
---


Le contrôle RichTextBox de WPF est assez puissant, et très pratique quand on a besoin d’accepter une saisie en texte riche. Cependant, l’une de ses fonctionnalités peut devenir problématique : l’utilisateur peut coller une image. Selon ce qu’on veut faire du texte saisi par l’utilisateur, ce n’est pas forcément souhaitable.

Quand j’ai cherché sur le web un moyen d’empêcher cela, les seules solutions que j’ai trouvées suggéraient d’intercepter la frappe de touches Ctrl-V, et de bloquer l’événement si le presse-papiers contient une image. Cette approche présente plusieurs problèmes:

- elle n’empêche pas l’utilisateur de coller via le menu contextuel
- elle ne fonctionne pas si le raccourci de la commande a été modifié
- elle n’empêche pas l’utilisateur d’insérer une image par glisser-déposer


Puisque cette solution ne me convenait pas, j’ai utilisé le site [.NET Framework Reference Source](http://referencesource.microsoft.com/) pour chercher un moyen d’intercepter l’opération de collage elle-même. J’ai suivi le code à partir de la propriété `ApplicationCommands.Paste`, et j’ai finalement trouvé l’événement attaché [DataObject.Pasting](https://msdn.microsoft.com/en-us/library/system.windows.dataobject.pasting.aspx). Ce n’est pas un endroit où j’aurais pensé à chercher, mais quand on y réfléchit, c’est finalement assez logique. Cet événement peut être utilisé pour intercepter une opération de collage ou de glisser-déposer, et permet au gestionnaire de l’événement de faire différentes choses:

- annuler purement et simplement l’opération
- changer le format de données qui sera collé depuis le presse-papiers
- remplacer le `DataObject` utilisé pour le collage


Dans mon cas, je voulais juste empêcher le collage ou le glisser-déposer d’une image, donc j’ai simplement annulé l’opération quand le `FormatToApply` était `"Bitmap"`, comme illustré ci-dessous.

XAML:

```xml
<RichTextBox DataObject.Pasting="RichTextBox1_Pasting" ... />
```

Code-behind:

```csharp
private void RichTextBox1_Pasting(object sender, DataObjectPastingEventArgs e)
{
    if (e.FormatToApply == "Bitmap")
    {
        e.CancelCommand();
    }
}
```

Bien sûr, il est également possible de gérer ça plus intelligemment. Par exemple, si le `DataObject` contient plusieurs formats, on pourrait créer un nouveau `DataObject` avec uniquement les formats acceptables. Comme ça l’utilisateur peut encore coller *quelque chose*, tant que ce n’est pas une image.

