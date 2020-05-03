---
layout: post
title: '[C# 5] Programmation asynchrone avec C# 5'
date: 2010-10-30T00:00:00.0000000
url: /2010/10/30/c-5-programmation-asynchrone-avec-c-5/
tags:
  - async
  - await
  - C# 5.0
categories:
  - C# 5.0
---


Depuis quelque temps, les spéculations allaient bon train sur les fonctionnalités de la future version 5 du langage C#… Très peu d’informations officielles avaient filtré à ce sujet, la seule chose plus ou moins certaine était l’introduction du concept de “compilateur en temps que service”, qui permettrait de tirer parti du compilateur à partir du code. A part ça, silence radio de la part de Microsoft…

Lors de la PDC jeudi dernier, un coin du voile a enfin été levé, mais pas du tout sur ce qu’on attendait ! Anders Hejlsberg, le créateur de C#, a bien consacré quelques minutes à la notion de “*compiler as a service*”, mais l’essentiel de sa présentation portait sur quelque chose de complètement différent : la programmation asynchrone en C#.

Il est bien sûr déjà possible d’effectuer des traitements asynchrones en C#, mais c’est généralement assez pénible et peu intuitif… On est souvent obligé de passer par des callbacks pour indiquer ce qui doit être exécuté à la fin du traitement asynchrone, et on se retrouve rapidement avec un code difficile à relire et à comprendre, et donc à maintenir. Pour une démonstration de ce problème, je vous invite à lire l’[excellent article d’Eric Lippert](http://blogs.msdn.com/b/ericlippert/archive/2010/10/27/continuation-passing-style-revisited-part-five-cps-and-asynchrony.aspx) à ce sujet, il explique ça beaucoup mieux que moi…

Cet article (et [la série](http://blogs.msdn.com/b/ericlippert/archive/tags/continuation+passing+style/) qu’il conclut) était en fait un prélude à l’annonce faite à la PDC : **C# 5 intègrera une nouvelle syntaxe permettant d’écrire du code asynchrone** de façon beaucoup plus naturelle, avec l’introduction de deux nouveaux mots-clés : `async` et `await`. Le code à écrire pour réaliser un traitement asynchrone sera quasiment identique à celui d’un traitement synchrone : toute la complexité sera masquée par cette nouvelle fonctionnalité du langage.

Puisqu’un exemple vaut mieux qu’un long discours, je vais reprendre l’exemple utilisé par Anders Hejlsberg pendant sa présentation, en le simplifiant un peu. Supposons qu’on veuille rechercher des titres de films par leur année de sortie. Pour simplifier, on utilisera le service OData de [Netflix](http://www.netflix.com/). Le code suivant effectue la recherche de façon synchrone, en récupérant les résultats 10 par 10 :

```
        private void btnSearch_Click(object sender, RoutedEventArgs e)
        {
            int year;
            if (!int.TryParse(txtYear.Text, out year))
            {
                MessageBox.Show("L'année saisie est incorrecte");
                return;
            }
            SearchMovies(year);
        }

        private void SearchMovies(int year)
        {
            var netflixUri = new Uri("http://odata.netflix.com/Catalog/");
            var catalog = new Netflix.NetflixCatalog(netflixUri);
            lstTitles.Items.Clear();
            int count = 0;
            int pageSize = 10;
            while (true)
            {
                var movies = SearchMoviesBatch(catalog, year, count, pageSize);
                if (movies.Length == 0)
                    break;
                foreach (var title in movies)
                {
                    lstTitles.Items.Add(title.Name);
                }
                count += movies.Length;
            }
        }

        private Title[] SearchMoviesBatch(NetflixCatalog catalog, int year, int count, int pageSize)
        {
            var query = from title in catalog.Titles
                            where title.ReleaseYear == year
                            orderby title.Name
                            select title;
            return query.Skip(count).Take(pageSize).ToArray();
        }
```

Ce code a le mérite d’être assez simple, mais il suffit de l’exécuter pour se rendre compte qu’il y a un problème : la récupération des résultats peut prendre un certain temps, pendant lequel l’interface reste figée. Il faut donc effectuer la recherche de façon asynchrone, pour que l’interface reste réactive. Voici une approche possible, avec la version actuelle de C# :

```
        private void SearchMoviesAsync(int year)
        {
            lstTitles.Items.Clear();
            Thread t = new Thread(() =>
            {
                var netflixUri = new Uri("http://odata.netflix.com/Catalog/");
                var catalog = new Netflix.NetflixCatalog(netflixUri);
                int count = 0;
                int pageSize = 10;
                while (true)
                {
                    var movies = SearchMoviesBatch(catalog, year, count, pageSize);
                    if (movies.Length == 0)
                        break;
                    foreach (var title in movies)
                    {
                        Dispatcher.Invoke(new Action(() => lstTitles.Items.Add(title.Name)));
                    }
                    count += movies.Length;
                }
            });
            t.Start();
        }
```

(Les deux autres méthodes sont inchangées)

On voit que le code commence déjà à être moins clair, à cause de l’expression lambda passée au constructeur du thread, et de l’utilisation de `Dispatcher.Invoke` pour mettre à jour l’interface graphique. Imaginez un peu ce que ça donnerait dans un scénario plus complexe, avec plusieurs tâches asynchrones interdépendantes (comme dans l’article d’Eric Lippert mentionné plus haut).

Avec la nouvelle syntaxe introduite par C# 5, voici comment on pourrait écrire ce code :

```
        private async void SearchMoviesAsync(int year)
        {
            var netflixUri = new Uri("http://odata.netflix.com/Catalog/");
            var catalog = new Netflix.NetflixCatalog(netflixUri);
            lstTitles.Items.Clear();
            int count = 0;
            int pageSize = 10;
            while (true)
            {
                var movies = await SearchMoviesBatchAsync(catalog, year, count, pageSize);
                if (movies.Length == 0)
                    break;
                foreach (var title in movies)
                {
                    lstTitles.Items.Add(title.Name);
                }
                count += movies.Length;
            }
        }

        private async Task<Title[]> SearchMoviesBatchAsync(NetflixCatalog catalog, int year, int count, int pageSize)
        {
            var query = from title in catalog.Titles
                        where title.ReleaseYear == year
                        orderby title.Name
                        select title;
            return await query.Skip(count).Take(pageSize).ToArrayAsync();
        }
```

Remarquez que les deux méthodes ont un nouveau modificateur `async`, qui indique qu’elles s’exécutent de façon asynchrone. Lors de l’appel à une autre méthode asynchrone, l’appel est précédé du mot-clé `await`. Lorsque la méthode `SearchMoviesAsync` est appelée, elle commence à s’exécuter normalement, jusqu’au mot-clé `await`. A partir de là, deux scénarios sont possibles

- soit l’appel à `SearchMoviesBatchAsync` se termine de façon synchrone, auquel cas l’exécution continue normalement
- soit il s’exécute de façon asynchrone, dans ce cas le contrôle est rendu à la méthode qui appelle `SearchMoviesAsync` (en l’occurrence `btnSearch_Click`). Quand l’appel à `SearchMoviesBatchAsync` se termine, l’exécution de `SearchMoviesAsync` reprend là où elle en était (de ce point de vue, `await` fonctionne un peu comme `yield return`)


Un peu comme pour les blocs itérateurs, le compilateur réécrit le code de la méthode en créant un delegate avec le code qui suit l’appel asynchrone, et appelle ce delegate quand la tâche asynchrone se termine. Remarquez d’ailleurs que ce delegate est appelé sur le même thread, celui du dispatcher en l’occurrence : on n’a donc pas besoin de `Dispatcher.Invoke` pour mettre à jour l’interface graphique.

En pratique, tout ce système se base sur la classe `Task` introduite dans .NET 4. Remarquez d’ailleurs que le type de retour de la méthode `SearchMoviesBatchAsync` est `Task<Title[]>`. Pourtant, quand on appelle cette méthode à partir de `SearchMoviesAsync`, on récupère bien un objet de type <font face="Courier New">Title[]</font>, et non `Task<Title[]>`. C’est l’autre effet du mot-clé `await` : il récupère le *résultat* d’une tâche une fois qu’elle est terminée.

Encore une chose : j’ai utilisé dans le code une méthode d’extension `ToArrayAsync`, voici son code :

```
        public static Task<T[]> ToArrayAsync<T>(this IQueryable<T> source)
        {
            return TaskEx.Run(() => source.ToArray());
        }
```

Voilà pour l’introduction à cette future nouvelle fonctionnalité de C#. J’espère que c’était à peu près compréhensible et que je n’ai pas dit trop de bêtises… tout n’est pas encore complètement clair dans ma tête![Clignement d&#039;œil](wlemoticon-winkingsmile.png). Pour en savoir plus, voici quelques liens utiles :

- Eric Lippert, encore lui, a commencé une série d’articles sur la programmation asynchrone en C# 5, voici les deux premiers :
    - [Asynchrony in C# 5, Part One](http://blogs.msdn.com/b/ericlippert/archive/2010/10/28/asynchrony-in-c-5-part-one.aspx)
    - [Asynchronous Programming in C# 5.0 part two: Whence await?](http://blogs.msdn.com/b/ericlippert/archive/2010/10/29/asynchronous-programming-in-c-5-0-part-two-whence-await.aspx)
- [La vidéo de la présentation d’Anders Hejlsberg à la PDC](http://player.microsoftpdc.com/Session/1b127a7d-300e-4385-af8e-ac747fee677a)
- [Asynchronous Programming for C# and Visual Basic](http://msdn.microsoft.com/fr-fr/vstudio/async.aspx), la page officielle où vous pourrez trouver des vidéos et articles sur le sujet, ainsi que le Visual Studio Async CTP que vous pouvez télécharger pour essayer vous-même.


Pour les adeptes de VB.NET, sachez que cette fonctionnalité sera aussi inclue dans la prochaine version de Visual Basic, ainsi que les itérateurs, qui n’existaient qu’en C# jusqu’à maintenant.

