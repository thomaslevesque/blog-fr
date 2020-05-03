---
layout: post
title: Afficher facilement une taille de fichier sous forme lisible par un humain
date: 2014-11-24T00:00:00.0000000
url: /2014/11/24/afficher-facilement-une-taille-de-fichier-sous-forme-lisible-par-un-humain/
tags:
  - byte
  - C#
  - octet
  - size
  - taille
categories:
  - Librairies
---


Si vous écrivez une application qui a un rapport avec la gestion de fichiers, vous aurez probablement besoin d’afficher la taille des fichiers. Mais si un fichier a une taille de 123456789 octets, ce n’est évidemment pas la valeur qu’il faudra afficher, car c’est difficile à lire, et l’utilisateur n’a généralement pas besoin de connaitre la taille à l’octet près. Vous allez plutôt afficher quelque chose comme 118 Mo.

Ca ne devrait a priori pas être très compliqué, mais en fait il y a différentes façons d’afficher une taille en octets… Par exemple, plusieurs conventions coexistent pour les unités et préfixes :

- La convention [SI](http://fr.wikipedia.org/wiki/Syst%C3%A8me_international_d%27unit%C3%A9s) (Système International d’Unités) utilise des multiples décimaux, basés sur des puissances de 10 : 1 kilooctet vaut 1000 octets, 1 mégaoctet vaut 1000 kilooctets, etc. Les préfixes sont ceux du système métrique (k, M, G, etc.).
- La convention [CEI](http://fr.wikipedia.org/wiki/CEI_60027-2)  (Commission Electrotechnique Internationale) utilise des multiples binaires, basés sur des puissances de 2 : 1 kibioctet vaut 1024 octets, 1 mébioctet vaut 1024 kibioctets, etc. Les préfixes sont Ki, Mi, Gi, etc., pour éviter la confusion avec le système métrique.
- Mais aucune de ces conventions n’est communément utilisée : la convention usuelle est d’utiliser des multiples binaires (1024), mais des préfixes décimaux (K, M, G, etc.).


Selon le contexte, on utilisera l’une ou l’autre de ces conventions. Je n’ai jamais vu la convention SI utilisée où que ce soit ; certaines applications (je l’ai vu dans VirtualBox par exemple) utilisent la convention CEI ; la plupart des applications et systèmes d’exploitation utilisent la convention usuelle. Vous pouvez lire cet article Wikipédia si vous voulez en savoir plus : [Préfixe binaire](http://fr.wikipedia.org/wiki/Pr%C3%A9fixe_binaire).

OK, alors supposons qu’on a choisi la convention usuelle pour l’instant. Maintenant, il faut décider quelle échelle utiliser : voulez-vous écrire 0,11 Go, 118 Mo, 120564 Ko ou 123456789 o ? Habituellement, on choisit l’échelle de façon à ce que la valeur affichée soit entre 1 et 1024.

Il y a encore quelques autres éléments à prendre en compte :

- Voulez-vous afficher des valeurs entières, ou inclure quelques chiffres après la virgule ?
- Y a-t-il une unité minimale à utiliser (par exemple, Windows n’affiche jamais des octets : un fichier d’1 octet est affiché comme 1 Ko) ?
- Comment la valeur doit-elle être arrondie ?
- Comment faut-il formater la valeur ?
- Pour les valeurs inférieures à 1 Ko, voulez vous utiliser le mot “octets”, ou juste le symbole “o” ?


### Bon, ça suffit! Où veux-tu en venir?

Comme vous pouvez le voir, afficher une taille en octets sous forme lisible par des humains n’est pas aussi évident qu’on aurait pu s’y attendre… J’ai eu à écrire du code pour le faire dans plusieurs applications, et j’ai fini par en avoir assez de refaire à chaque fois, donc j’ai créé une librairie qui s’efforce de couvrir tous les cas d’utilisation. Je l’ai appelée [HumanBytes](https://github.com/thomaslevesque/HumanBytes), pour des raisons qui devraient être évidentes… Elle est également disponible sous forme de [package NuGet](https://www.nuget.org/packages/HumanBytes).

Son utilisation est assez simple. Elle est basée sur la classe `ByteSizeFormatter`, qui expose des propriétés pour contrôler la façon dont la valeur est formatée :

```
var formatter = new ByteSizeFormatter
{
    Convention = ByteSizeConvention.Binary,
    DecimalPlaces = 1,
    NumberFormat = "#,##0.###",
    MinUnit = ByteSizeUnit.Kilobyte,
    MaxUnit = ByteSizeUnit.Gigabyte,
    RoundingRule = ByteSizeRounding.Closest,
    UseFullWordForBytes = true,
};

var f = new FileInfo("TheFile.jpg");
Console.WriteLine("The size of '{0}' is {1}", f, formatter.Format(f.Length));
```

Cependant, dans la plupart des cas, vous voudrez simplement utiliser les paramètres par défaut. Vous pouvez le faire facilement grâce à la méthode d’extension `Bytes` :

```
var f = new FileInfo("TheFile.jpg");
Console.WriteLine("The size of '{0}' is {1}", f, f.Length.Bytes());
```

Cette méthode renvoie une instance de la structure `ByteSize`, dont la méthode `ToString` formate la valeur avec le formateur par défaut. Vous pouvez changer les paramètres du formateur par défaut via la propriété statique `ByteSizeFormatter.Default`.

### A propos de la localisation

Toutes les langues n’utilisent pas le même symbole pour “octet”, et bien sûr le mot “octet” lui-même est différent d’une langue à l’autre. Pour l’instant, HumanBytes ne supporte que l’anglais et le français ; si vous voulez ajouter le support d’une autre langue, n’hésitez pas à forker le projet, ajouter votre traduction, et faire une pull request. Il n’y a que 3 termes à traduire, donc ça ne devrait pas prendre trop longtemps ![Winking smile](wlEmoticon-winkingsmile.png).

