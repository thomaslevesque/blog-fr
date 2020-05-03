---
layout: post
title: Méthodes C# dans les en-têtes de diff git
date: 2017-02-10T00:00:00.0000000
url: /2017/02/10/methodes-c-dans-les-en-tetes-de-diff-git/
tags:
  - async
  - C#
  - csharp
  - diff
  - git
  - gitattributes
  - xfuncname
categories:
  - Uncategorized
---


Si vous utilisez git en ligne de commande, vous aurez peut-être remarqué que les diffs indiquent souvent la signature de la méthode dans l'en-tête du bloc (la ligne qui commence par `@@`), comme ceci :

```diff
diff --git a/Program.cs b/Program.cs
index 655a213..5ae1016 100644
--- a/Program.cs
+++ b/Program.cs
@@ -13,6 +13,7 @@ static void Main(string[] args)
         Console.WriteLine("Hello World!");
         Console.WriteLine("Hello World!");
         Console.WriteLine("Hello World!");
+        Console.WriteLine("blah");
     }
```

C'est très pratique pour savoir où vous vous trouvez quand vous regardez un diff.

Git a quelques patterns d'expressions régulières prédéfinis pour détecter les méthodes dans quelques langages, y compris C#; ils sont définis dans [`userdiff.c`](https://github.com/git/git/blob/d7dffce1cebde29a0c4b309a79e4345450bf352a/userdiff.c#L140). Mais par défaut, ces patterns ne sont pas utilisés... il faut dire à git quelles extensions de fichier sont associées à quel langage. Cela peut être spécifié dans un fichier `.gitattributes` à la racine de votre repo :

```
*.cs    diff=csharp
```

Cela fait, `git diff` devrait afficher une sortie similaire à l'exemple ci-dessus.

Est-ce que ça suffit ? Presque. En fait, les patterns pour C# ont été ajoutés à git il y a longtemps, et C# a pas mal évolué depuis. Certains nouveaux mots-clés qui peuvent maintenant apparaître dans une signature de méthode ne sont pas reconnus par le pattern prédéfini, par exemple `async` ou `partial`. C'est assez agaçant, parce que quand du code a changé dans une méthode asynchrone, l'en-tête du bloc dans le diff indique la signature de la précédente méthode non-asynchrone, ou la ligne où la classe est déclarée, ce qui prête à confusion.

Mon premier réflexe a été de soumettre une pull request sur Github pour ajouter les mots-clés manquants ; cependant j'ai vite réalisé que le [repo de git sur Github](https://github.com/git/git) est juste un miroir et n'accepte pas de pull requests... Le [processus de contribution](https://github.com/git/git/blob/master/Documentation/SubmittingPatches) consiste à envoyer un patch à la mailing list de git, avec une longue et ennuyeuse liste de règles à respecter. Ce processus m'a semblé si laborieux que j'ai abandonné l'idée. Franchement, je ne sais pas pourquoi git utilise un processus de contribution aussi obsolète et compliqué, ça ne fait que décourager les contributeurs occasionnels. Mais c'est un peu hors-sujet, alors passons et voyons si on peut résoudre le problème autrement.

Heureusement, les patterns prédéfinis peuvent être redéfinis dans la configuration de git. Pour définir le pattern de signature de fonction pour C#, il faut définir le paramètre `diff.csharp.xfuncname` dans votre fichier de configuration git :

```
[[diff "csharp"]]
  xfuncname = ^[ \\t]*(((static|public|internal|private|protected|new|virtual|sealed|override|unsafe|async|partial)[ \\t]+)*[][<>@.~_[:alnum:]]+[ \\t]+[<>@._[:alnum:]]+[ \\t]*\\(.*\\))[ \\t]*$
```

Comme vous pouvez le voir, c'est le même pattern que dans `userdiff.c`, avec les backslashes échappés et les mots-clés manquants ajoutés. Avec ce pattern, `git diff` affiche maintenant la signature correcte pour les méthodes asynchrones :

```diff
diff --git a/Program.cs b/Program.cs
index 655a213..5ae1016 100644
--- a/Program.cs
+++ b/Program.cs
@@ -31,5 +32,6 @@ static async Task FooAsync()
         Console.WriteLine("Hello world");
         Console.WriteLine("Hello world");
         Console.WriteLine("Hello world");
+        await Task.Delay(100);
     }
 }
```

Ça m'a pris un moment de trouver comment le faire marcher, alors j'espère que ça vous sera utile !

