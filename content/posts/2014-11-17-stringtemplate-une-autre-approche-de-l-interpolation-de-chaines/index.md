---
layout: post
title: 'StringTemplate: une autre approche de l’interpolation de chaines'
date: 2014-11-17T00:00:00.0000000
url: /2014/11/17/stringtemplate-une-autre-approche-de-l-interpolation-de-chaines/
tags:
  - C# 6
  - interpolation de chaines
  - localisation
  - NString
  - StringTemplate
categories:
  - Uncategorized
---


Avec la version 6 de C# qui approche, il y a beaucoup de discussions sur CodePlex et ailleurs à propos de l’interpolation de chaines. Pas très étonnant, puisqu’il s’agit d’une des fonctionnalités majeures de cette version… Au cas où vous auriez vécu dans une grotte ces derniers mois et n’en auriez pas entendu parler, l’interpolation de chaines est un moyen d’insérer des expressions C# à l’intérieur d’une chaine de caractère, de façon à ce qu’elles soient évaluées lors de l’exécution et remplacées par leurs valeurs. En gros, vous écrivez quelque chose comme ça :

```
string text = $"{p.Name} est né le {p.DateOfBirth:D}";
```

Et le compilateur le transforme en ceci :

```
string text = String.Format("{0} est né le {1:D}", p.Name, p.DateOfBirth);
```

**Note**: la syntaxe présentée ci-dessus correspond aux [dernières notes de conception sur cette fonctionnalité](http://roslyn.codeplex.com/discussions/570292). Elle peut encore changer d’ici à la sortie finale, et la preview actuelle de VS2015 utilise encore une syntaxe différente : `“\{p.Name} est né le \{p.DateOfBirth:D}”`.

J’adore cette fonctionnalité. Elle va être extrêmement pratique pour des choses comme le logging, la génération d’URL ou de requêtes, etc. Je l’utiliserai certainement beaucoup, surtout maintenant que Microsoft a écouté les retours de la communauté et a inclus un moyen de personnaliser la façon dont les expressions dans la chaine sont évaluées (regardez la partie à propos de `IFormattable` dans les [notes de conception](http://roslyn.codeplex.com/discussions/570292)).

Cependant, quelque chose me chiffonne : puisque les chaines interpolées sont interprétées par le compilateur, elles *doivent* être codées en dur ; on ne peut pas les extraire dans des ressources pour la localisation. Cela signifie que cette fonctionnalité n’est pas utilisable pour la localisation, et qu’on est obligés de continuer à utiliser des marqueurs numériques dans les chaines localisées.

Mais est-ce vraiment inévitable ?

Depuis quelques années, j’utilise un [moteur d’interpolation de chaines](https://github.com/thomaslevesque/NString#stringtemplate) que j’ai créé, qui s’utilise de la même façon que `String.Format`, mais avec des marqueurs nommés plutôt que des numéros. Il prend une chaine de format, et un objet avec des propriétés qui correspondent aux noms des marqueurs :

```
string text = StringTemplate.Format("{Name} est né le {DateOfBirth:D}", new { p.Name, p.DateOfBirth });
```

Bien sûr, si vous avez déjà un objet avec les propriétés que vous voulez inclure dans la chaine, vous pouvez simplement passer cet objet directement :

```
string text = StringTemplate.Format("{Name} est né le {DateOfBirth:D}", p);
```

Le résultat est exactement ce à quoi on pourrait s’attendre : les marqueurs sont remplacés par les valeurs des propriétés correspondantes.

En quoi est-ce mieux que `String.Format` ?

- C’est beaucoup plus lisible; un marqueur nommé indique immédiatement quelle valeur ira à cet emplacement
- On est moins susceptible de se tromper : pas besoin de faire attention à l’ordre des valeurs à formater
- Quand on extrait les chaines de format dans des ressources pour la localisation, le traducteur voit un nom dans le marqueur, pas un numéro. Cela donne plus de contexte à la chaine, et permet de comprendre plus facilement à quoi la chaine finale va ressembler.


Notez que vous pouvez utiliser les mêmes spécificateurs de format que dans `String.Format`. La classe `StringTemplate` analyse la chaine de format et la transforme en une chaine compatible avec `String.Format`, extrait les valeurs des propriétés dans un tableau, et appelle `String.Format`.

Bien sûr, analyser la chaine et extraire les valeurs des propriétés par réflexion à chaque fois serait très inefficace, il y a donc des optimisations :

- chaque chaine de format distincte est analysée une seule fois, et le résultat de l’analyse est mis en cache pour être réutilisé plus tard.
- pour chaque propriété utilisée dans une chaine de format, un delegate accesseur est généré et mis en cache, pour éviter de faire appel à la réflexion à chaque fois.


Cela signifie que la première fois que vous utilisez une chaine de format données, il y a un coût lié à l’analyse et à la génération des delegates, mais les utilisations ultérieures de la même chaine de format sont beaucoup plus rapides.

La classe `StringTemplate` fait partie d’une librairie nommée [NString](https://github.com/thomaslevesque/NString), qui contient également des méthodes d’extension pour faciliter la manipulation de chaines. La librairie est une PCL qui peut être utilisée avec toutes les variantes de .NET à l’exception de Silverlight 5. Un paquet NuGet est disponible [ici](https://www.nuget.org/packages/NString/).

