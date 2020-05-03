---
layout: post
title: Quoi de neuf dans FakeItEasy 3.0.0 ?
date: 2017-02-26T00:00:00.0000000
url: /2017/02/26/quoi-de-neuf-dans-fakeiteasy-3-0-0/
tags:
  - FakeItEasy
  - mocking
  - tests unitaires
categories:
  - Uncategorized
---


[FakeItEasy](https://fakeiteasy.github.io/) est un framework de mocking populaire pour .NET, avec une API intuitive et facile à utiliser. Depuis environ un an, je suis un des principaux développeurs de FakeItEasy, avec [Adam Ralph](https://github.com/adamralph/) and [Blair Conrad](https://github.com/blairconrad/). Ça a été un vrai plaisir de travailler avec eux, et je me suis éclaté !

**Aujourd'hui j'ai le plaisir d'annoncer la sortie de [FakeItEasy 3.0.0](https://github.com/FakeItEasy/FakeItEasy/releases/tag/3.0.0), avec le support de .NET Core et quelques fonctionnalités utiles.**

Voyons ce que cette nouvelle version apporte !

## Support de .NET Core

En plus de .NET 4+, FakeItEasy supporte maintenant .NET Standard 1.6, vous pouvez donc l'utiliser dans vos projets .NET Core.

Notez qu'en raison de certaines limitations de .NET Standard 1.x, il y a quelques différences mineures avec la version .NET 4 de FakeItEasy :

- Les fakes ne sont pas sérialisables avec la sérialisation binaire
- Les fakes "auto-initialisés" (self-initializing fakes) ne sont pas supportés (c'est-à-dire `fakeService = A.Fake<IService>(options => options.Wrapping(realService).RecordedBy(recorder))`).


Un immense merci aux personnes qui ont rendu possible le support de .NET Core :

- [Jonathon Rossi](https://github.com/jonorossi), qui maintient le projet [Castle.Core](https://github.com/castleproject/Core). FakeItEasy s'appuie beaucoup sur Castle.Core, il n'aurait donc pas été possible de supporter .NET Core sans que Castle.Core le supporte aussi.
- [Jeremy Meng](https://github.com/jeremymeng) de Microsoft, qui a fait l'essentiel du gros-œuvre pour porter FakeItEasy et Castle.Core vers .NET Core.


## Analyseur

### Support de VB.NET

L'[analyseur FakeItEasy](http://fakeiteasy.readthedocs.io/en/stable/analyzer/), qui avertit en cas d'usage incorrect de la librairie, supporte maintenant VB.NET en plus de C#.

## Nouvelles fonctionnalités et améliorations

### Syntaxe améliorée pour configurer des appels successifs au même membre

Quand on configure les appels vers un fake, cela crée des règles qui sont "empilées" les unes sur les autres, ce qui fait qu'on peut écraser une règle précédemment configurée. Combiné avec la possibilité de spécifier le nombre de fois qu'une règle doit s'appliquer, cela permet de configurer des choses comme "renvoie 42 deux fois, puis lance une exception". Jusqu'ici, pour faire cela il fallait configurer les appels dans l'ordre inverse, ce qui n'était pas très intuitif et obligeait à répéter la spécification de l'appel :

```csharp
A.CallTo(() => foo.Bar()).Throws(new Exception("oops"));
A.CallTo(() => foo.Bar()).Returns(42).Twice();
```

FakeItEasy 3.0.0 introduit une nouvelle syntaxe pour rendre cela plus simple :

```csharp
A.CallTo(() => foo.Bar()).Returns(42).Twice()
    .Then.Throws(new Exception("oops"));
```

Notez que si vous ne spécifiez pas le nombre de fois que la règle doit s'appliquer, elle s'appliquera indéfiniment jusqu'à ce qu'elle soit écrasée. Par conséquent, on ne peut utiliser `Then` qu'après `Once()`, `Twice()` ou `NumberOfTimes(...)`.

C'est un breaking change au niveau de l'API, dans la mesure où la forme des interfaces de configuration a changé, mais à moins que vous ne manipuliez explicitement ces interfaces, vous ne devriez pas être affecté.

### Support automatique pour l'annulation

Quand une méthode accepte un paramètre de type `CancellationToken`, elle doit généralement lancer une exception si on lui passe un token qui est déjà annulé. Jusqu'à maintenant, ce comportement devait être configuré manuellement. Dans FakeItEasy 3.0.0, les méthodes d'un fake lanceront une `OperationCanceledException` par défaut si on les appelle avec un token déjà annulé. Les méthodes asynchrones renverront une tâche annulée.

Techniquement, c'est également un breaking change, mais la plupart des utilisateurs ne devraient pas être affectés.

### Lancer une exceptionde façon asynchrone

FakeItEasy permet de configurer une méthode pour qu'elle lance une exception grâce à la méthode `Throws`. Mais pour les méthodes asynchrones, il y a en fait deux façons de signaler une erreur :

- lancer une exception de façon synchrone, avant même de renvoyer une tâche (c'est ce que fait `Throws`)
- renvoyer une tâche échouée (cela devait être fait manuellement jusqu'à maintenant)


Dans certains cas la différence peut être importante pour l'appelant, s'il n'`await` pas directement la méthode asynchrone. FakeItEasy introduit une méthode `ThrowsAsync` pour configurer une méthode pour qu'elle renvoie une tâche échouée :

```csharp
A.CallTo(() => foo.BarAsync()).ThrowsAsync(new Exception("foo"));
```

### Configuration des setters de propriétés sur les fakes non-naturels

Les fakes non-naturels (c'est-à-dire `Fake<T>`) ont maintenant une méthode `CallsToSet`, qui fait la même chose que `A.CallToSet` sur les fakes naturels :

```csharp
var fake = new Fake<IFoo>();
fake.CallsToSet(foo => foo.Bar).To(0).Throws(new Exception("The value of Bar can't be 0"));
```

### API améliorée pour spécifier des attributs supplémentaires

L'API pour spécifier des attributs supplémentaires sur les fakes n'était pas très pratique; il fallait créer une collection de `CustomAttributeBuilder`s, qui eux-mêmes devaient être créés en spécifiant le constructeur et les valeurs des arguments. La méthode `WithAdditionalAttributes` a été supprimée dans FakeItEasy 3.0.0 et remplacée par une méthode plus simple `WithAttributes` qui accepte des expressions :

```csharp
var foo = A.Fake<IFoo>(x => x.WithAttributes(() => new FooAttribute()));
```

C'est un breaking change.

## Autres changements notables

### Dépréciation des fakes auto-initialisés (self-initializing fakes)

Les fakes "auto-initialisés" permettent d'enregistrer les appels faits sur un vrai objet, et de faire rejouer les résultats obtenus par un fake (voir [l'article de Martin Fowler](https://martinfowler.com/bliki/SelfInitializingFake.html) à ce sujet). Cette fonctionnalité était utilisée par *très* peu de gens, et n'avait pas vraiment sa place dans la librairie principale de FakeItEasy. Elle est donc dépréciée et sera retirée dans une version future. Nous envisageons de fournir une fonctionnalité équivalente en tant que package distinct.

### Corrections de bugs

- Créer plusieurs fakes d'un même type avec des attributs supplémentaires différents génère maintenant plusieurs types de fake distincts. ([#436](https://github.com/FakeItEasy/FakeItEasy/issues/436))
- Tous les appels non-void tentent maintenant de renvoyer un Dummy par défaut, même après avoir été reconfigurés par `Invokes` ou `DoesNothing` ([#830](https://github.com/FakeItEasy/FakeItEasy/issues/830))


* * *

La liste complète des changements est disponible dans les [notes de versions](https://github.com/FakeItEasy/FakeItEasy/releases/tag/3.0.0).

Les autres personnes ayant contribué à cette version sont :

- Artem Zinenko - [@ar7z1](https://github.com/ar7z1)
- Christian Merat - [@cmerat](https://github.com/cmerat)
- [@thunderbird55](https://github.com/thunderbird55)


Un grand merci à eux !

