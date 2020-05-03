---
layout: post
title: Test unitaires asynchrones avec NUnit
date: 2015-02-02T00:00:00.0000000
url: /2015/02/02/test-unitaires-asynchrones-avec-nunit/
tags:
  - async
  - C#
  - nunit
  - tests unitaires
categories:
  - Tests unitaires
---


Récemment, mon équipe et moi avons commencé à écrire des tests unitaires pour une application qui utilise beaucoup de code asynchrone. Nous avons utilisé NUnit (2.6) parce que nous le connaissions déjà bien, mais nous ne l’avions encore jamais utilisé pour tester du code asynchrone.

Supposons que le système à tester soit cette très intéressante classe `Calculator` :

```
    public class Calculator
    {
        public async Task<int> AddAsync(int x, int y)
        {
            // simulate long calculation
            await Task.Delay(100).ConfigureAwait(false);
            // the answer to life, the universe and everything.
            return 42;
        }
    }
```

*(Indice: ce code contient un bug… 42 n’est pas toujours la réponse. Ça m’a fait un choc quand j’ai appris ça!)*

Et voici un test unitaire pour la méthode `AddAsync` :

```
        [Test]
        public async void AddAsync_Returns_The_Sum_Of_X_And_Y()
        {
            var calculator = new Calculator();
            int result = await calculator.AddAsync(1, 1);
            Assert.AreEqual(2, result);
        }
```
``
## `async void` vs. `async Task`

Avant même de lancer ce test, je me suis dit : *Ça ne va pas marcher! une méthode `async void` va retourner immédiatement sur le premier `await`, NUnit va donc croire que le test est terminé alors que l’assertion n’a pas été exécutée, et le test va donc passer même si l’assertion échoue.* J’ai donc changé la signature de la méthode en `async Task`, en me croyant très malin d’avoir évité ce piège…

```
        [Test]
        public async Task AddAsync_Returns_The_Sum_Of_X_And_Y()
```

Comme prévu, le test a échoué, ce qui confirme que NUnit sait gérer les tests asynchrones. J’ai corrigé la classe `Calculator`, et je n’y ai plus pensé. Jusqu’au jour où j’ai remarqué qu’un collègue écrivait ses tests avec `async void`. J’ai donc commencé à lui expliquer pourquoi ça ne pouvait pas marcher, et j’ai essayé de le lui démontrer en ajoutant une assertion qui échouerait… et à ma grande surprise, le test a échoué, prouvant que j’avais tort !

Etant d’une nature curieuse, j’ai aussitôt commencé à investiguer… Ma première idée a été de vérifier le `SynchronizationContext` courant, et en effet, j’ai vu que NUnit l’avant remplacé par une instance de `NUnit.Framework.AsyncSynchronizationContext`. Cette classe maintient une file des continuations qui sont postées dessus. Après que la méthode `async void` retourne (c’est-à-dire la première fois qu’on `await` une tâche qui n’est pas encore terminée), NUnit appelle la méthode `WaitForPendingOperationsToComplete`, qui exécute toute les continuations de la file, jusqu’à ce que celle-ci soit vide. C’est seulement là que le test sera considéré comme terminé.

La morale de cette histoire est donc que vous *pouvez* écrire des tests `async void` avec NUnit 2.6. Cela fonctionne aussi avec les delegates passés à `Assert.Throws`, qui peuvent avoir le modificateur `async`. Cela étant dit, ce n’est pas parce que vous pouvez le faire que c’est forcément une bonne idée… Les frameworks de test unitaire n’ont pas tous le même support pour cela, et **la prochaine version de NUnit (la 3.0, encore en alpha),**[**ne supportera pas les tests async void**](https://github.com/nunit/nunit/blob/d922adae5cd30ad5544ee693f6ae6177722e3569/src/NUnitFramework/framework/Internal/AsyncInvocationRegion.cs#L76)``**.**

Donc, à moins que vous ne comptiez rester sur NUnit 2.6.4 ad vitam æternam, il vaut probablement toujours utiliser `async Task` dans vos tests unitaires.

