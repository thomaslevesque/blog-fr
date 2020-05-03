---
layout: post
title: 'Piège: utiliser var et async ensemble'
date: 2016-06-24T00:00:00.0000000
url: /2016/06/24/piege-utiliser-var-et-async-ensemble/
tags:
  - async
  - bug
  - C#
  - ReSharper
  - testing
  - unit
categories:
  - Uncategorized
---


Il y a quelques jours au bureau, je suis tombé sur un bug assez sournois dans notre application principale. Le code semblait assez innocent, et à première vue je ne voyais vraiment pas ce qui n’allait pas… Le code était similaire à ceci:

```csharp
public async Task<bool> BookExistsAsync(int id)
{
    var store = await GetBookStoreAsync();
    var book = store.GetBookByIdAsync(id);
    return book != null;
}

// Pour donner le contexte, voici les types et méthodes utilisés dans BookExistsAsync:

private Task<IBookStore> GetBookStoreAsync()
{
    // ...
}


public interface IBookStore
{
    Task<Book> GetBookByIdAsync(int id);
    // ...
}

public class Book
{
    public int Id { get; set; }
    // ...
}
```

La méthode `BookExistsAsync` renvoie toujours `true`. Voyez-vous pourquoi ?

Regardez cette ligne :

```csharp
var book = store.GetBookByIdAsync(id);
```

Vite, sans réfléchir, quel est le type de `book` ? Si vous avez répondu `Book`, regardez de plus près : c’est `Task<Book>`. Il manque le `await` ! Et une méthode `async` renvoie toujours une tâche non nulle, donc `book` n’est jamais nul.

Quand vous avez une méthode `async` sans `await`, le compilateur vous avertit, mais en l’occurrence il y a un `await` sur la ligne précédente. La seule chose qu’on fait avec `book` est de vérifier qu’il n’est pas `null` ; puisque `Task<T>` est un type référence, il n’y rien de suspect dans le fait de le comparer à `null`. Donc le compilateur ne voit rien d’anormal ; l’analyseur statique de code (ReSharper dans mon cas) ne voit rien d’anormal ; et le pauvre cerveau humain qui fait la revue de code ne voit rien d’anormal non plus… Évidemment, le problème aurait facilement pu être détecté avec une couverture de tests adéquate, mais malheureusement cette méthode n’était pas couverte.

Alors, comment éviter ce type d’erreur ? En arrêtant d’utiliser `var` et en spécifiant toujours les types explicitement ? Mais *j’aime* `var`, je l’utilise presque partout ! D’ailleurs, je crois que c’est la première fois que je vois un bug lié à l’utilisation de `var`. Je ne suis vraiment pas prêt à l’abandonner…

Idéalement, j’aurais aimé que ReSharper détecte le problème; peut-être qu’il devrait considérer que toutes les méthodes qui renvoient une `Task` sont implicitement `[NotNull]`, sauf mention contraire. En attendant, je n’ai pas de solution miracle pour ce problème ; faites juste attention quand vous appelez une méthode asynchrone, et écrivez des tests unitaires !

