---
layout: post
title: Créer un conteneur qui crée automatiquement des objets factices avec Unity et FakeItEasy
date: 2015-06-16T00:00:00.0000000
url: /2015/06/16/crer-un-conteneur-qui-cre-automatiquement-des-objets-factices-avec-unity-et-fakeiteasy/
tags:
  - C#
  - FakeItEasy
  - injection de dépendances
  - mocking
  - Unity
categories:
  - Tests unitaires
---


L’écriture de tests unitaires peut parfois être un peu rébarbative, surtout quand on teste des classes qui ont des dépendances complexes. Heureusement, certains outils rendent cela un peu plus facile. J’utilise beaucoup [FakeItEasy](https://github.com/FakeItEasy/FakeItEasy) ces derniers temps ; c’est un framework de mock pour .NET très facile à utiliser. Il a une API simple et élégante, basée sur les génériques et les expressions lambda, et c’est un vrai plaisir de l’utiliser. Ça a été une bouffée d’air frais par rapport au vieux RhinoMocks que j’utilisais jusqu’ici.

Mais malgré toutes les qualités de FakeItEasy, l’enregistrement des fakes (objets factices) pour les dépendances de la classe testée est encore un peu laborieuse. Ce serait bien si le conteneur IoC pouvait nous créer automatiquement des fakes à la volée, non ? Le code suivant :



```
var container = new UnityContainer();
 
// Setup dependencies
var fooProvider = A.Fake<IFooProvider>();
container.RegisterInstance(fooProvider);
var barService = A.Fake<IBarService>();
container.RegisterInstance(barService);
var bazManager = A.Fake<IBazManager>();
container.RegisterInstance(bazManager);
 
var sut = container.Resolve<SystemUnderTest>();
```

Pourrait être réduit à ceci :

```
var container = new UnityContainer();
 
// This will cause the container to provide fakes for all dependencies
container.AddNewExtension<AutoFakeExtension>();
 
var sut = container.Resolve<SystemUnderTest>();
```

Eh bien, c’est en fait assez facile à faire avec Unity. Unity n’est pas vraiment le conteneur IoC le plus à la mode, mais il est bien supporté, facile à utiliser, et extensible. J’ai créé l’extension suivante pour permettre le scénario décrit plus haut :

```
public class AutoFakeExtension : UnityContainerExtension
{
    protected override void Initialize()
    {
        Context.Strategies.AddNew<AutoFakeBuilderStrategy>(UnityBuildStage.PreCreation);
    }
     
    private class AutoFakeBuilderStrategy : BuilderStrategy
    {
        private static readonly MethodInfo _fakeGenericDefinition;
     
        static AutoFakeBuilderStrategy()
        {
            _fakeGenericDefinition = typeof(A).GetMethod("Fake", Type.EmptyTypes);
        }
         
        public override void PreBuildUp(IBuilderContext context)
        {
            if (context.Existing == null)
            {
                var type = context.BuildKey.Type;
                if (type.IsInterface || type.IsAbstract)
                {
                    var fakeMethod = _fakeGenericDefinition.MakeGenericMethod(type);
                    var fake = fakeMethod.Invoke(null, new object[0]);
                    context.PersistentPolicies.Set<ILifetimePolicy>(new ContainerControlledLifetimeManager(), context.BuildKey);
                    context.Existing = fake;
                    context.BuildComplete = true;
                }
            }
            base.PreBuildUp(context);
        }
    }
}
```

Quelques commentaires sur ce code :

- La logique est un peu rudimentaire (on génère un fake uniquement pour les interfaces et classes abstraites), mais elle peut facilement être ajustée selon les besoins.
- Le vilain hack de réflexion est dû au fait que FakeItEasy n’a pas de surcharge non générique pour la méthode `A.Fake` qui accepte un `Type` en paramètre. Nul n’est parfait…
- La durée de vie du fake est définie à “container controlled” (c’est-à-dire qu’elle a la durée de vie du conteneur), parce que si on a besoin de configurer les appels sur le fake, il faut qu’on puisse accéder à la même instance que celle qui est injectée dans la classe testée :


```
var fooProvider = container.Resolve<IFooProvider>();
A.CallTo(() => fooProvider.GetFoo(42)).Returns(new Foo { Id = 42, Name = “test” });
```

Remarquez que si on enregistre explicitement une dépendance dans le conteneur, elle sera utilisée en priorité et aucun fake ne sera créé. On peut donc utiliser cette extension tout en gardant la possibilité de spécifier certaines dépendances manuellement :

```
var container = new UnityContainer();
container.AddNewExtension<AutoFakeExtension>();
container.RegisterType<IFooProvider, TestFooProvider>();
var sut = container.Resolve<SystemUnderTest>();
```

Bien sûr, cette extension pourrait facilement être modifiée pour utiliser un framework de mock autre que FakeItEasy. Je suppose que le même principe pourrait aussi être appliqué à d’autres conteneurs IoC, pourvu qu’ils aient des points d’extension appropriés.

### Et AutoFixture dans tout ça ?

Avant que vous ne posiez la question : oui, je connais [AutoFixture](https://github.com/AutoFixture/AutoFixture). C’est une très bonne librairie, et j’ai aussi essayé de l’utiliser, avec un certain succès. Le code est d’ailleurs très similaire aux exemples ci-dessus. La principale raison pour laquelle je n’ai pas continué à l’utiliser est qu’AutoFixture n’est pas vraiment un conteneur IoC (bien qu’il fasse certaines des choses que fait un conteneur IoC), et je préfère utiliser le même mécanisme d’injection de dépendance dans mes tests unitaires que dans le code applicatif. De plus, je n’étais pas très à l’aise avec la façon dont AutoFixture gère les propriétés quand il crée une instance de la classe à tester. Par défaut, il initialise toutes les propriétés publiques avec des fakes de leur type ; c’est très bien dans le cas des propriétés utilisées pour l’injection de dépendance, mais ça n’a pas de sens pour les autres propriétés. Je sais qu’il est possible d’inhiber ce comportement pour des propriétés spécifiques, mais il faut le faire manuellement au cas par cas ; cela ne prend pas en compte les attributs spécifiques au conteneur IoC, comme `[Dependency]`. Au final, j’ai trouvé plus simple d’utiliser mon extension Unity personnalisée, dont je comprends et maitrise bien le comportement.

