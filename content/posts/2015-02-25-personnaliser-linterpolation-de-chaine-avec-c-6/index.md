---
layout: post
title: Personnaliser l’interpolation de chaine avec C# 6
date: 2015-02-25T00:00:00.0000000
url: /2015/02/25/personnaliser-linterpolation-de-chaine-avec-c-6/
tags:
  - C# 6
  - iformattable
  - roslyn
  - string interpolation
categories:
  - Uncategorized
---


L’une des principales nouveautés de C# 6 est l’interpolation de chaines de caractères, qui permet d’écrire ce genre de chose :

```
string text = $"{p.Name} was born on {p.DateOfBirth:D}";
```

Un aspect peu connu de cette fonctionnalité est qu’une chaine interpolée peut être traitée soit comme un `String`, soit comme un `IFormattable`, selon le contexte. Quand elle est convertie en `IFormattable`, cela crée un objet `FormattableString` qui implémente l’interface et expose :

- la chaine de format, avec les valeurs remplacées par des marqueurs numériques (compatible avec `String.Format`)
- les valeurs pour les marqueurs


La méthode `ToString()` de cet objet appelle simplement `String.Format(format, values)`. Mais il y a aussi une surcharge qui accepte un `IFormatProvider`, et c’est là que ça devient intéressant, parce que cela permet de personnaliser la façon dont les valeurs sont formatées. Il n’est peut-être pas évident de voir en quoi c’est utile, donc laissez moi vous montrer quelques exemples…

### Spécifier la culture

Pendant la conception de la fonctionnalité d’interpolation de chaines, il y a eu un débat assez vif pour décider s’il fallait utiliser la culture courante ou la culture neutre (“invariant”) pour formater les valeurs; il y avait de bons arguments des deux côtés, mais au final il a été décidé d’utiliser la culture courante, par souci de cohérence avec `String.Format` et des APIs similaires qui utilisent la [mise en forme composite](https://msdn.microsoft.com/fr-fr/library/txafckwd.aspx). Utiliser la culture courante est pertinent quand on utilise l’interpolation de chaines pour construire des chaines qui seront affichée dans l’interface utilisateur ; mais il y a aussi des scénarios où on veut construire des chaines qui seront utilisées dans des APIs ou protocoles (URLs, requêtes SQL…), et dans ces cas là il faut généralement utiliser la culture neutre.

C# 6 fournit un moyen facile de faire cela, en tirant parti de la conversion en `IFormattable`. Il suffit de créer une méthode comme celle-ci :

```
static string Invariant(FormattableString formattable)
{
    return formattable.ToString(CultureInfo.InvariantCulture);
}
```

Et vous pouvez ensuite l’utiliser comme suit:

```
string text = Invariant($"{p.Name} was born on {p.DateOfBirth:D}");
```

Les valeurs dans la chaine interpolée seront formatées avec la culture neutre, et non plus avec la culture courante.

### Construire des URLs

Voici un exemple plus avancé. L’interpolation de chaines est un moyen pratique de construire des URLs, mais si on inclut des chaines arbitraires dans l’URL, il faut prendre soin de les encoder pour ne pas avoir de caractères invalides dans l’URL. Un interpolateur de chaine personnalisé peut le faire pour nous; il faut juste créer un `IFormatProvider` personnalisé qui s’occupera d’encoder les valeurs. L’implémentation n’était pas évidente au premier abord, mais après quelques tâtonnements je suis arrivé à ceci :

```
class UrlFormatProvider : IFormatProvider
{
    private readonly UrlFormatter _formatter = new UrlFormatter();

    public object GetFormat(Type formatType)
    {
        if (formatType == typeof(ICustomFormatter))
            return _formatter;
        return null;
    }

    class UrlFormatter : ICustomFormatter
    {
        public string Format(string format, object arg, IFormatProvider formatProvider)
        {
            if (arg == null)
                return string.Empty;
            if (format == "r")
                return arg.ToString();
            return Uri.EscapeDataString(arg.ToString());
        }
    }
}
```

Ce formateur peut être utiliser comme ceci :

```
static string Url(FormattableString formattable)
{
    return formattable.ToString(new UrlFormatProvider());
}

...

string url = Url($"http://foobar/item/{id}/{name}");
```

Cela va correctement encoder les valeurs de `id` et `name` de façon à ce que l’’URL générée ne contienne que des caractères valides.

*Aparté: Avez-vous remarqué le if `(format == "r")`? C’est un spécificateur de format personnalisé qui indique que la valeur ne doit pas être encodé (“r” pour “raw”). Pour l’utiliser, il suffit de l’inclure dans la chaine de format comme ceci : `{id:r}`. Cela empêchera l’encodage de `id`.*

### Construire des requêtes SQL

On peut faire quelque chose de similaire pour les requêtes SQL. Bien sûr, intégrer des valeurs directement dans une requête est une mauvaise pratique bien connue, pour des raison de sécurité et de performance (il faut utiliser des requêtes paramétrées); mais pour un développement “à l’arrache”, ça peut parfois être utile. Et puis c’est une bonne illustration de cette fonctionnalité. Pour intégrer des valeurs dans une requête SQL, il faut :

- encadrer les chaines entre des apostrophes, et échapper les apostrophes à l’intérieur des chaines en les doublant
- formater les dates en fonction de ce que le SGBD attend (généralement MM/dd/yyyy)
- formater les nombres selon la culture neutre
- remplacer les valeurs nulles par le littéral `NULL`.


(il y a probablement d’autres choses à prendre en compte, mais ce sont les plus évidentes).

On peut utiliser la même approche que pour les URLs, et créer un `SqlFormatProvider` :

```
class SqlFormatProvider : IFormatProvider
{
    private readonly SqlFormatter _formatter = new SqlFormatter();

    public object GetFormat(Type formatType)
    {
        if (formatType == typeof(ICustomFormatter))
            return _formatter;
        return null;
    }

    class SqlFormatter : ICustomFormatter
    {
        public string Format(string format, object arg, IFormatProvider formatProvider)
        {
            if (arg == null)
                return "NULL";
            if (arg is string)
                return "'" + ((string)arg).Replace("'", "''") + "'";
            if (arg is DateTime)
                return "'" + ((DateTime)arg).ToString("MM/dd/yyyy") + "'";
            if (arg is IFormattable)
                return ((IFormattable)arg).ToString(format, CultureInfo.InvariantCulture);
            return arg.ToString();
        }
    }
}
```

On peut ensuite utiliser ce formateur comme ceci :

```
static string Sql(FormattableString formattable)
{
    return formattable.ToString(new SqlFormatProvider());
}

...

string sql = Sql($"insert into items(id, name, creationDate) values({id}, {name}, {DateTime.Now})");
```

De cette façon les valeurs seront correctement formatées pour générer une requête SQL valide.

### Utiliser l’interpolation de chaines quand on cible des versions plus anciennes de .NET

Comme c’est souvent le cas avec les fonctionnalités du langage qui exploitent des types du .NET Framework, il est possible d’utiliser cette fonctionnalité avec des versions plus anciennes de .NET qui n’ont pas la classe `FormattableString` ; il suffit de créer la classe soi-même dans le namespace approprié. En fait, il y a en l’occurrence deux classes à implémenter : `FormattableString` et `FormattableStringFactory`. [Jon Skeet était apparemment très pressé d’essayer](https://twitter.com/jonskeet/status/569973023064887296), et il a déjà [donné un exemple](https://gist.github.com/jskeet/9d297d0dc013d7a557ee) avec le code pour ces classes :

```
using System;

namespace System.Runtime.CompilerServices
{
    public class FormattableStringFactory
    {
        public static FormattableString Create(string messageFormat, params object[] args)
        {
            return new FormattableString(messageFormat, args);
        }

        public static FormattableString Create(string messageFormat, DateTime bad, params object[] args)
        {
            var realArgs = new object[args.Length + 1];
            realArgs[0] = "Please don't use DateTime";
            Array.Copy(args, 0, realArgs, 1, args.Length);
            return new FormattableString(messageFormat, realArgs);
        }
    }
}

namespace System
{
    public class FormattableString
    {
        private readonly string messageFormat;
        private readonly object[] args;

        public FormattableString(string messageFormat, object[] args)
        {
            this.messageFormat = messageFormat;
            this.args = args;
        }
        public override string ToString()
        {
            return string.Format(messageFormat, args);
        }
    }
}
```

C’est la même approche qui permettait d’utiliser Linq en ciblant .NET 2.0 ([LinqBridge](http://www.albahari.com/nutshell/linqbridge.aspx)) ou les [attributs d’infos de l’appelant quand on cible .NET 4.0 ou plus ancien](http://www.thomaslevesque.fr/2012/06/13/utiliser-les-informations-de-lappelant-de-c-5-quand-on-cible-une-version-plus-ancienne-du-net-framework/). Bien sûr, ça nécessite quand même le compilateur C# 6 pour fonctionner…

### Conclusion

La conversion de chaines interpolées en `IFormattable` avait [déjà été mentionnée](https://roslyn.codeplex.com/discussions/570614) il y a quelque temps, mais n’était pas encore implémentée dans Visual Studio 2015 CTP 5. La [CTP 6 qui vient d’être publiée](http://blogs.msdn.com/b/visualstudio/archive/2015/02/23/visual-studio-2015-ctp-6-and-team-foundation-server-2015-ctp-released.aspx) embarque une nouvelle version du compilateur qui inclut cette fonctionnalité, vous pouvez donc commencer à jouer avec ! Cette fonctionnalité rend l’interpolation de chaine très flexible, et je suis sûr que les gens vont trouver toutes sortes de cas d’utilisation auxquels je n’avais pas pensé.

Vous pouvez trouver le code des exemples ci-dessus [sur GitHub](https://github.com/thomaslevesque/blog-code-samples/tree/master/TestStringInterpolation).

