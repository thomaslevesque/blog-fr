---
layout: post
title: Propriétés et éléments MSBuild partagés avec Directory.Build.props
date: 2017-09-20T00:00:00.0000000
url: /2017/09/20/proprietes-et-elements-msbuild-partages-avec-directory-build-props/
tags:
  - .NET
  - .net core
  - build
  - csproj
  - msbuild
  - Visual Studio
categories:
  - Uncategorized
---


Pour être honnête, je n'ai jamais vraiment aimé MSBuild jusqu'à récemment. Les fichiers de projet générés par Visual Studio étaient immondes, l'essentiel de leur contenu était redondant, il fallait décharger les projets pour les éditer, c'était mal documenté... Mais avec l'avènement de .NET Core et du nouveau format de projet, plus léger, MSBuild est devenu un bien meilleur outil.

MSBuild 15 a introduit une nouvelle fonctionnalité assez sympa : les imports implicites (je ne sais pas si c'est le nom officiel, mais c'est celui que j'utiliserai). En gros, vous pouvez créer un fichier nommmé `Directory.Build.props` n'importe où dans votre solution, et il sera automatiquement importé par tous les projets sous le répertoire qui contient ce fichier. Cela permet de partager très facilement des propriétés et éléments communs entre les projets. Cette fonctionnalité est décrite en détail sur [cette page](https://docs.microsoft.com/en-us/visualstudio/msbuild/customize-your-build).

Par exemple, si vous voulez partager certaines métadonnées entre plusieurs projets, créer simplement un fichier `Directory.Build.props` dans le dossier parent de vos projets :

```xml


  
    1.2.3
    John Doe
  
```

On peut aussi faire des choses plus intéressantes, comme activer et configurer StyleCop pour tous les projets :

```xml


  
    
    $(MSBuildThisFileDirectory)MyRules.ruleset
  

  
    
    
    
    
    
  
```

*Notez que la variable `$(MSBuildThisFileDirectory)` fait référence au répertoire contenant le fichier MSBuild courant. Une autre variable utile est `$(MSBuildProjectDirectory)`, qui fait référence au répertoire du projet en cours de génération.*

MSBuild cherche le fichier `Directory.Build.props` en partant du répertoire du projet et en remontant les dossiers jusqu'à ce qu'il trouve un fichier correspondant, puis s'arrête de chercher. Dans certains cas, il peut être utile de définir des propriétés communes à tous les projets, et d'en ajouter d'autres qui ne s'appliquent qu'à un sous-répertoire. Pour faire cela, il faut que le fichier `Directory.Build.props` le plus "profond" importe explicitement celui du répertoire parent :

- (rootDir)/Directory.build.props:


```xml


  
  
  
```

- (rootDir)/tests/Directory.build.props:


```xml


  
  

  
  
  
```

La documentation mentionne une autre approche, utilisant la fonction `GetPathOfFileAbove`, mais cela ne semblait pas fonctionner quand j'ai essayé... De toute façon, je pense qu'il est plus simple d'utiliser un chemin relatif, on risque moins de se tromper.

Utiliser les imports implicites apporte quelques avantages :

- des fichiers de projet plus petits, puisque les propriétés et éléments identiques peuvent être factorisés dans des fichiers communs
- un seul point de référence : si tous les projets référencent le même package NuGet, la version à référencer est définie à un seul endroit; il n'est plus possible d'avoir des incohérences.


Cette approche a cependant un inconvénient : Visual Studio n'a pas la notion de l'origine d'une variable ou d'un élément, donc si vous changez une propriété ou une référence de package dans l'IDE (via les pages de propriétés du projet ou le gestionnaire de packages NuGet), elle sera modifiée dans le fichier de projet lui-même, et non dans le fichier `Directory.Build.props`. De mon point de vue, ce n'est pas un gros problème, parce que j'ai pris l'habitude d'éditer les projets manuellement plutôt que d'utiliser l'IDE, mais ça peut être gênant pour certaines personnes.

Si vous voulez un exemple réel de l'utilisation de cette technique, jetez un oeil au repository de [FakeItEasy](https://github.com/FakeItEasy/FakeItEasy), où nous utilisons plusieurs fichiers `Directory.Build.props` pour garder les projets propres et concis.

Notez que vous pouvez également créer un fichier `Directory.Build.targets`, suivant les mêmes principes, pour définir des cibles communes.

