---
layout: post
title: Casse-tête C# n°2
date: 2015-08-02T00:00:00.0000000
url: /2015/08/02/casse-tte-c-n2/
tags:
  - C#
  - casse-tête
categories:
  - Casse-têtes
---


Encore un petit casse-tête basé sur un problème que j’ai rencontré au boulot…

Regardez ce morceau de code :

```
Console.WriteLine($"x > y is {x > y}");
Console.WriteLine($"!(x <= y) is {!(x <= y)}");
```

Comment faudrait-il déclarer x et y pour que le programme produise la sortie (apparemment illogique) suivante ?

```
x > y is False
!(x <= y) is True
```

