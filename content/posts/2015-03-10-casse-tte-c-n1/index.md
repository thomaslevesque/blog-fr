---
layout: post
title: Casse-tête C# n°1
date: 2015-03-10T00:00:00.0000000
url: /2015/03/10/casse-tte-c-n1/
tags:
  - C#
  - casse-tête
categories:
  - Casse-têtes
---


J’adore résoudre des casse-têtes en C#; je pense que c’est un excellent moyen d’approfondir sa connaissance du langage. Et en plus, c’est amusant !

Je viens de penser à celui-ci :

```
static void Test(out int x, out int y)
{
    x = 42;
    y = 123;
    Console.WriteLine (x == y);
}
```

Que pensez-vous que ce code affiche ? Pouvez-vous en être sûr ? Postez votre réponse dans les commentaires !

J’essaierai de poster plus de casse-têtes à l’avenir, si j’en trouve d’autres.

