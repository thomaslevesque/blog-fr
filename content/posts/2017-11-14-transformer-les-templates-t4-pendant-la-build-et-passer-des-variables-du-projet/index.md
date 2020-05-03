---
layout: post
title: Transformer les templates T4 pendant la build, et passer des variables du projet
date: 2017-11-14T00:00:00.0000000
url: /2017/11/14/transformer-les-templates-t4-pendant-la-build-et-passer-des-variables-du-projet/
tags:
  - build
  - code generation
  - msbuild
  - T4
  - Visual Studio
categories:
  - Uncategorized
---


[T4 (Text Template Transformation Toolkit)](https://docs.microsoft.com/en-us/visualstudio/modeling/code-generation-and-t4-text-templates) est un excellent outil pour générer du code ; on peut, par exemple, créer des classes POCO à partir des tables d'une base de données, générer du code répétitif, etc. Dans Visual Studio, les fichiers T4 (extension .tt) sont associés au custom tool `TextTemplatingFileGenerator`, qui transforme un template pour générer un fichier de sortie à chaque fois qu'on enregistre le template. Mais il arrive que ce ne soit pas suffisant, et qu'on souhaite regénérer les sorties des templates à chaque build. C'est assez facile à mettre en œuvre, mais il y a quelques écueils à éviter.

## Transformer les templates lors du build

Si votre projet est un csproj ou vbproj "classique" (c'est-à-dire *pas* un projet .NET Core "SDK-style"), les choses sont assez simples et bien documentées sur [cette page](https://docs.microsoft.com/en-us/visualstudio/modeling/code-generation-in-a-build-process).

Déchargez votre projet, et ouvrez le dans l'éditeur. Ajoutez le `PropertyGroup` suivant vers le début du fichier :

```xml

    
    15.0
    $(MSBuildExtensionsPath32)\Microsoft\VisualStudio\v$(VisualStudioVersion)
    
    true
    
    true
    
    false
```

Et ajoutez l'`Import` suivant à la fin, après l'import de `Microsoft.CSharp.targets` ou `Microsoft.VisualBasic.targets` :

```xml

```

Rechargez votre projet, et le tour est joué. Générer le projet devrait maintenant transformer les templates et regénérer leur sortie.

## Projets "SDK-style"

Si vous utilisez le nouveau format de projet du SDK .NET Core (souvent appelé de façon informelle projet "SDK-style"), l'approche décrite ci-dessus nécessite un petit changement pour fonctionner. C'est parce que le fichier de cibles par défaut (`Sdk.targets` dans le SDK .NET Core) est maintenant importé implicitement à la toute fin du projet, il n'est donc pas possible d'importer les cibles T4 après les cibles par défaut. Du coup la variable `BuildDependsOn`, qui est modifiée par les cibles T4, est écrasée par les cibles par défaut, et la cible `TransformAll` ne s'exécute plus avant la cible `Build`.

Heureusement, il y a un workaround : vous pouvez importer les cibles par défaut explicitement, et importer les cibles T4 après celles-ci :

```xml

```

Notez que cela causera un avertissement MSBuild dans la sortie de la build (MSB4011), parce que `Sdk.targets` est importé deux fois ; cet avertissement peut être ignoré sans risque.

## Passer des variables MSBuild aux templates

À un moment donné, il est possible que la logique de génération de code devienne trop complexe pour rester entièrement dans le template T4. Vous pourriez vouloir en extraire une partie dans un assembly utilitaire, qui sera référencé depuis le template comme ceci :

```
<#@ assembly name="../CodeGenHelper/bin/Release/net462/CodeGenHelper.dll" #>
```

Mais spécifier le chemin de l'assembly de cette façon n'est pas très pratique... par exemple, si vous êtes actuellement sur la configuration `Debug`, la version `Release` de CodeGenHelper.dll ne sera pas forcément à jour. Heureusement, le custom tool `TextTemplatingFileGenerator` de Visual Studio supporte l'utilisation de variables MSBuild du projet, il est donc possible de faire ceci :

```
<#@ assembly name="$(SolutionDir)/CodeGenHelper/bin/$(Configuration)/net462/CodeGenHelper.dll" #>
```

Les variables `$(SolutionDir)` et `$(Configuration)` seront remplacées par leurs valeurs respectives. Si vous enregistrez le template, il sera bien transformé en utilisant l'assembly CodeGenHelper.dll. Pratique !

Mais il y a un hic... Si vous avez configuré votre projet pour transformer les templates lors de la build, comme décrit plus haut, la build va maintenant échouer, avec une erreur comme celle-ci :


> System.IO.FileNotFoundException: Could not find a part of the path 'C:\Path\To\The\Project\$(SolutionDir)\CodeGenHelper\bin\$(Configuration)\net462\CodeGenHelper.dll'.


Vous avez remarqué les variables `$(SolutionDir)` et `$(Configuration)` dans le chemin ? Elles n'ont pas été remplacées ! C'est parce que la cible MSBuild qui transforme les templates et le custom tool `TextTemplatingFileGenerator` n'utilisent pas le même moteur de transformation de texte. Et malheureusement, celui utilisé par MSBuild ne supporte pas nativement les variables MSBuild... un comble !

Cependant tout n'est pas perdu. Il suffit de spécifier explicitement les variables que vous voulez passer en temps que paramètres T4. Éditez à nouveau votre fichier projet, et créez un nouveau `ItemGroup` avec les éléments suivants :

```xml

    
        $(SolutionDir)
        False
    
    
        $(Configuration)
        False
    
```

L'attribut `Include` est le nom du paramètre tel qu'il sera passé au moteur de transformation de texte. L'élément `Value`, comme son nom l'indique, est la valeur du paramètre. Et l'élément `Visible` permet d'éviter que l'élément `T4ParameterValues` n'apparaisse comme élément du projet dans l'explorateur de solution.

Avec ce changement, la build devrait à nouveau transformer les templates correctement.

Pour résumer, gardez à l'esprit que le custom tool `TextTemplatingFileGenerator` et la cible MSBuild de transformation de texte ont des mécanismes différents pour passer des variables aux templates :

- `TextTemplatingFileGenerator` supporte *uniquement* les variables MSBuild du projet
- MSBuild supporte *uniquement* `T4ParameterValues`


Donc si vous utilisez des variables dans votre template, et que vous voulez pouvoir transformer le template quand vous l'enregistrez dans Visual Studio *et* quand vous générez le projet, les variables doivent être définies à la fois comme variables MSBuild et comme `T4ParameterValues`.

