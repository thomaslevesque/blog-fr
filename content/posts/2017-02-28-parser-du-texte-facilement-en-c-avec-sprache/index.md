---
layout: post
title: Parser du texte facilement en C# avec Sprache
date: 2017-02-28T00:00:00.0000000
url: /2017/02/28/parser-du-texte-facilement-en-c-avec-sprache/
tags:
  - parser
  - parser combinator
  - parsing
  - sprache
categories:
  - Uncategorized
---


Il y a quelques jours, j'ai découvert un petit bijou : [Sprache](https://github.com/sprache/sprache). Le nom signifie "langage" en allemand. C'est une librairie très élégante et facile à utiliser pour créer des analyseurs de texte, à l'aide de [parser combinators](https://en.wikipedia.org/wiki/Parser_combinator), qui sont une technique très courante en programmation fonctionnelle. Le concept théorique peut sembler un peu effrayant, mais comme vous allez le voir dans un instant, Sprache rend ça très accessible.

## Analyse syntaxique

L'analyse syntaxique (parsing) est une tâche très courante, mais qui peut être laborieuse et où il est facile de faire des erreurs. Il existe de nombreuses approches :

- analyse manuelle basée sur `Split`, `IndexOf`, `Substring` etc.
- expressions régulières
- parser codé à la main qui scanne les tokens dans une chaine
- parser généré avec ANTLR ou un outil similaire
- et certainement beaucoup d'autres qui m'échappent...


Aucune de ces options n'est très séduisante. Pour les cas simples, découper la chaine ou utiliser une regex peut suffire, mais ça devient vite ingérable pour des grammaires plus complexes. Construire un vrai parser à la main pour une grammaire non-triviale est loin d'être facile. ANTLR nécessite Java, un peu de connaissance, et se base sur de la génération de code, ce qui complique le process de build.

Heureusement, [Sprache](https://github.com/sprache/sprache) offre une alternative très intéressante. Il fournit de nombreux parsers et combinateurs prédéfinis qu'on peut utiliser pour définir une grammaire. Examinons pas à pas un cas concret : analyser le challenge dans le header `WWW-Authenticate` d'une réponse HTTP (j'ai dû faire un parser à la main pour ça récemment, et j'aurais aimé connaître Sprache à ce moment-là).

## La grammaire

Le header `WWW-Authenticate` est envoyé par un serveur HTTP dans une réponse 401 (Non autorisé) pour indiquer au client comment s'authentifier :

```
# Challenge Basic
WWW-Authenticate: Basic realm="FooCorp"

# Challenge OAuth 2.0 après l'envoi d'un token expiré
WWW-Authenticate: Bearer realm="FooCorp", error=invalid_token, error_description="The access token has expired"
```

Ce qu'on veut parser est le challenge, c'est-à-dire la valeur du header. Nous avons donc un mécanisme d'authentification ou "scheme" (`Basic`, `Bearer`), suivi d'un ou plusieurs paramètres (paires nom-valeur). Ça semble assez simple, on pourrait sans doute juste découper selon les `','`, puis selon les `'='` pour obtenir les valeurs... mais les guillemets compliquent les choses, car une chaine entre guillemets peut contenir les caractères `','` ou `'='`. De plus, les guillemets sont optionnels si la valeur du paramètre est un simple token, donc on ne peut pas compter sur le fait que les guillemets seront présents (ou pas). Clairement, si on veut parser ça de façon fiable, il va falloir regarder les specs de plus près...

Le header `WWW-Authenticate` est décrit en détails dans la [RFC-2617](https://tools.ietf.org/html/rfc2617). La grammaire ressemble à ceci, sous une forme que la spec appelle "forme Backus-Naur augmentée" (voir [RFC 2616 §2.1](https://tools.ietf.org/html/rfc2616#section-2.1)) :

```
# from RFC-2617 (HTTP Basic and Digest authentication)

challenge      = auth-scheme 1*SP 1#auth-param
auth-scheme    = token
auth-param     = token "=" ( token | quoted-string )

# from RFC2616 (HTTP/1.1)

token          = 1*<any CHAR except CTLs or separators>
separators     = "(" | ")" | "<" | ">" | "@"
               | "," | ";" | ":" | "\" | <">
               | "/" | "[" | "]" | "?" | "="
               | "{" | "}" | SP | HT
quoted-string  = ( <"> *(qdtext | quoted-pair ) <"> )
qdtext         = <any TEXT except <">>
quoted-pair    = "\" CHAR
```

Nous avons donc quelques règles de grammaire, voyons comment on peut les encoder en C# à l'aide de Sprache, et les utiliser pour analyser le challenge.

## Parser les tokens

Commençons par les parties les plus simples de la grammaire : les tokens ("jetons"). Un token est défini comme un ou plusieurs caractères qui ne sont ni des caractères de contrôle, ni des séparateurs.

Nous allons définir nos règles dans une classe `Grammar`. Commençons pour définir certaines classes de caractères :

```csharp
static class Grammar
{
    private static readonly Parser<char> SeparatorChar =
        Parse.Chars("()<>@,;:\\\"/[]?={} \t");

    private static readonly Parser<char> ControlChar =
        Parse.Char(Char.IsControl, "Control character");

}
```

- Chaque règle est déclarée comme un `Parser<T>` ; puisque ces règles valident des caractères seuls, elles sont de type `Parser<char>`.
- La classe `Parse` de Sprache expose des primitives d'analyse et des combinateurs.
- `Parse.Chars` valide n'importe quel caractère de la chaine spécifiée, on l'utilise pour spécifier la liste des caractères de séparation.
- La surcharge de `Parse.Char` qu'on utilise ici prend un prédicat qui sera appelé pour valider un caractère, et une description de cette classe de caractères. Ici on utilise la méthode `System.Char.IsControl` comme prédicat pour identifier les caractères de contrôle.


Définissons maintenant une classe de caractères `TokenChar`, qui correspond aux caractères qui peuvent être inclus dans un token. Selon la RFC, il s'agit de n'importe quel caractère qui n'appartient pas aux deux classes précédemment définies :

```csharp
    private static readonly Parser<char> TokenChar =
        Parse.AnyChar
            .Except(SeparatorChar)
            .Except(ControlChar);
```

- `Parse.AnyChar`, comme son nom l'indique, valide n'importe quel caractère.
- `Except` permet de spécifier des exceptions, c'est à dire des règles qui ne doivent *pas* valider le caractère.


Enfin, un token est une séquence d'un ou plusieurs de ces caractères :

```csharp
    private static readonly Parser<string> Token =
        TokenChar.AtLeastOnce().Text();
```

- Un token est une chaine, donc la règle pour un token est de type `Parser<string>`.
- `AtLeastOnce()` signifie une ou plusieurs répétitions, et puisque `TokenChar` est un `Parser<char>`, `AtLeastOnce()` renvoie un `Parser<IEnumerable<char>>`.
- `Text()` combine la séquence de caractères en une chaine, et renvoie donc un `Parser<string>`


Nous voilà donc capables de parser un token. Mais ce n'est qu'un premier pas, il nous reste encore pas mal de travail...

## Parser les chaines entre guillemets

La RFC définit une chaine entre guillemets (quoted string) comme une séquence de :

- un guillemet qui ouvre la chaine
- n'importe quel nombre de l'un de ces éléments :
    - un "qdtext", c'est-à-dire n'importe quel caractère sauf un guillemet
    - un "quoted pair", c'est-à-dire n'importe quel caractère précédé d'un backslash (c'est utilisé pour échapper les guillemets à l'intérieur d'une chaine)
- un guillemet qui ferme la chaine


Écrivons donc les règles pour "qdtext" et "quoted pair" :

```csharp
    private static readonly Parser<char> DoubleQuote = Parse.Char('"');
    private static readonly Parser<char> Backslash = Parse.Char('\\');

    private static readonly Parser<char> QdText =
        Parse.AnyChar.Except(DoubleQuote);

    private static readonly Parser<char> QuotedPair =
        from _ in Backslash
        from c in Parse.AnyChar
        select c;
```

La règle `QdText` se passe d'explication, mais `QuotedPair` est plus intéressante... Comme vous pouvez le voir, ça ressemble à une requête Linq : c'est comme ça qu'on spécifie une séquence avec Sprache. Cette requête-là signifie : *prendre un backslash (qu'on nomme `_` parce qu'on va l'ignorer) suivi de n'importe quel caractère nommé `c`, et renvoyer juste `c`* (les "quoted pairs" ne sont pas vraiment des séquences d'échappement comme en C, Java ou C#, donc `"\n"` n'est pas interprété comme "nouvelle ligne", mais simplement comme `"n"`).

On peut donc maintenant écrire la règle pour une chaine entre guillemets :

```csharp
    private static readonly Parser<string> QuotedString =
        from open in DoubleQuote
        from text in QuotedPair.Or(QdText).Many().Text()
        from close in DoubleQuote
        select text;
```

- La méthode `Or` indique un choix entre deux parsers. `QuotedPair.Or(QdText)` essaie de valider une "quoted pair", et si cela échoue, il essaie de valider un "qdtext" à la place.
- `Many()` indique un nombre quelconque de répétitions.
- `Text()` combine les caractères en une chaine.
- on sélectionne juste `text`, car on n'a plus besoin des guillemets (ils ne servaient qu'à délimiter la chaine).


Nous avons maintenant toutes les briques de bases, on va donc pouvoir passer à des règles de plus haut niveau.

### Parser les paramètres du challenge

Un challenge est constitué d'un scheme d'authentification suivi d'un ou plusieurs paramètres. Le scheme est trivial (c'est juste un token), donc commençons par parser les paramètres.

Bien que la RFC n'ait pas de règle nommée pour ça, définissons-nous une règle pour les valeurs des paramètres. La valeur peut être soit un token, soit une chaine entre guillemets :

```csharp
    private static readonly Parser<string> ParameterValue =
        Token.Or(QuotedString);
```

Puisqu'un paramètre est une élément composite (nom et valeur), déclarons une classe pour le représenter :

```csharp
class Parameter
{
    public Parameter(string name, string value)
    {
        Name = name;
        Value = value;
    }
    
    public string Name { get; }
    public string Value { get; }
}
```

Le `T` de `Parser<T>` n'est pas limité aux caractères ou aux chaines, ça peut être n'importe quel type. La règle pour parser les paramètres sera donc de type `Parser<Parameter>` :

```csharp
    private static readonly Parser<char> EqualSign = Parse.Char('=');

    private static readonly Parser<Parameter> Parameter =
        from name in Token
        from _ in EqualSign
        from value in ParameterValue
        select new Parameter(name, value);
```

Ici on prend un token (le nom du paramètre), suivi du signe `'='`, suivi d'une valeur de paramètre, et on combine le nom et la valeur en une instance de `Parameter`.

Analysons maintenant une séquence d'un ou plusieurs caractères. Les paramètres sont séparés par des virgules, avec des caractères d'espacement optionnels avant et après la virgule (chercher "#rule" dans la [RFC 2616 §2.1](https://tools.ietf.org/html/rfc2616#section-2.1)). La grammaire pour les listes autorise plusieurs virgules successives sans éléments entre elles, par exemple `item1 ,, item2,item3, ,item4`, donc la règle pour le séparateur de liste peut être écrite comme ceci :

```csharp
    private static readonly Parser<char> Comma = Parse.Char(',');

    private static readonly Parser<char> ListDelimiter =
        from leading in Parse.WhiteSpace.Many()
        from c in Comma
        from trailing in Parse.WhiteSpace.Or(Comma).Many()
        select c;
```

On valide juste la première virgule, le reste peut être n'importe quel nombre de virgules et de caractères d'espacement. On renvoie la virgule parce qu'il faut bien renvoyer quelque chose, mais on ne l'utilisera pas (dans un langage fonctionnel on aurait pu renvoyer le type `unit` à la place).

On peut maintenant analyser une séquence de paramètres comme ceci :

```csharp
    private static readonly Parser<Parameter[]> Parameters =
        from first in Parameter.Once()
        from others in (
            from _ in ListDelimiter
            from p in Parameter
            select p).Many()
        select first.Concat(others).ToArray();
```

Mais c'est un peu alambiqué... heureusement Sprache fournit une approche plus facile avec la méthode `DelimitedBy` :

```csharp
    private static readonly Parser<Parameter[]> Parameters =
        from p in Parameter.DelimitedBy(ListDelimiter)
        select p.ToArray();
```

## Parser le challenge

On y est presque. On a maintenant tout ce qu'il nous faut pour parser le challenge complet. Déclarons d'abord une classe pour le représenter :

```csharp
class Challenge
{
    public Challenge(string scheme, Parameter[] parameters)
    {
        Scheme = scheme;
        Parameters = parameters;
    }
    public string Scheme { get; }
    public Parameter[] Parameters { get; }
}
```

Et on peut enfin écrire la règle globale :

```csharp
    public static readonly Parser<Challenge> Challenge =
        from scheme in Token
        from _ in Parse.WhiteSpace.AtLeastOnce()
        from parameters in Parameters
        select new Challenge(scheme, parameters);
```

Remarquez que j'ai déclaré cette règle comme publique, contrairement aux autres : c'est la seule qu'on a besoin d'exposer.

## Utiliser le parseur

Notre parseur est terminé, il n'y a plus qu'à l'utiliser, ce qui est assez simple :

```csharp
void ParseAndPrintChallenge(string input)
{
    var challenge = Grammar.Challenge.Parse(input);
    Console.WriteLine($"Scheme: {challenge.Scheme}");
    Console.WriteLine($"Parameters:");
    foreach (var p in challenge.Parameters)
    {
        Console.WriteLine($"- {p.Name} = {p.Value}");
    }
}
```

Avec le challenge OAuth 2.0 de l'exemple précédent, ce code produit la sortie suivante :

```
Scheme: Bearer
Parameters:
- realm = FooCorp
- error = invalid_token
- error_description = The access token has expired
```

S'il y a une erreur de syntaxe dans le texte en entrée, la méthode `Parse` lancera une `ParseException` avec un message décrivant où et pourquoi l'analyse a échoué. Par exemple, si j'enlève l'espace entre "Bearer" et "realm", j'obtiens l'erreur suivante :


> Parsing failure: unexpected '='; expected whitespace (Line 1, Column 12); recently consumed: earerrealm


Vous trouverez le code complet de cet article [ici](https://gist.github.com/thomaslevesque/d8ee28be1cf383a3f8aaf39cee776f92).

## Conclusion

Comme vous le voyez, il est très facile avec Sprache de parser un texte complexe. Le code n'est pas particulièrement concis, mais il est complètement déclaratif ; il n'y pas de boucles, pas de conditions, pas de variables temporaires, pas d'état... Cela rend le code très facile à comprendre, et il peut facilement être comparé à la définition originale de la grammaire pour s'assurer de sa conformité. Sprache fournit aussi de bons retours en cas d'erreur, ce qui est assez difficile à implémenter dans un parseur écrit à la main.

