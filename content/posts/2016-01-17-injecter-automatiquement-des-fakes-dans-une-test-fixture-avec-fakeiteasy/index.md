---
layout: post
title: Injecter automatiquement des fakes dans une test fixture avec FakeItEasy
date: 2016-01-17T00:00:00.0000000
url: /2016/01/17/injecter-automatiquement-des-fakes-dans-une-test-fixture-avec-fakeiteasy/
tags:
  - C#
  - FakeItEasy
  - injection de dépendances
  - mocking
  - tests unitaires
categories:
  - Tests unitaires
---


Aujourd’hui j’aimerais partager avec vous une fonctionnalité sympa de [FakeItEasy](http://fakeiteasy.github.io/) que j’ai découverte récemment.

Quand on écrit des tests unitaires pour une classe qui a des dépendances, on a généralement besoin de créer des objets factices (fakes) pour les dépendances et de les injecter manuellement dans la classe testée (SUT, System Under Test), ou d’utiliser un conteneur IoC pour enregistrer les dépendances factices et construire le SUT. C’est un peu laborieux, et il y a quelques mois j’avais créé [une extension Unity pour créer automatiquement les fakes](/2015/06/16/crer-un-conteneur-qui-cre-automatiquement-des-objets-factices-avec-unity-et-fakeiteasy/) et rendre tout ça plus facile. Mais maintenant je réalise que FakeItEasy offre une solution encore meilleure : déclarer simplement les dépendances et le SUT comme des propriétés de la fixture, et appeler `Fake.InitializeFixture` sur la fixture pour les initialiser. Voilà à quoi ça ressemble :

```
public class BlogManagerTests
{
    [Fake] public IBlogPostRepository BlogPostRepository { get; set; }
    [Fake] public ISocialNetworkNotifier SocialNetworkNotifier { get; set; }
    [Fake] public ITimeService TimeService { get; set; }
 
    [UnderTest] public BlogManager BlogManager { get; set; }
 
    public BlogManagerTests()
    {
        Fake.InitializeFixture(this);
    }
 
    [Fact]
    public void NewPost_should_add_blog_post_to_repository()
    {
        var post = A.Dummy();
 
        BlogManager.NewPost(post);
 
        A.CallTo(() => BlogPostRepository.Add(post)).MustHaveHappened();
    }
 
    [Fact]
    public void PublishPost_should_update_post_in_repository_and_publish_link_to_social_networks()
    {
        var publishDate = DateTimeOffset.Now;
        A.CallTo(() => TimeService.Now).Returns(publishDate);
 
        var post = A.Dummy();
 
        BlogManager.PublishPost(post);
 
        Assert.Equal(BlogPostStatus.Published, post.Status);
        Assert.Equal(publishDate, post.PublishDate);
 
        A.CallTo(() => BlogPostRepository.Update(post)).MustHaveHappened();
        A.CallTo(() => SocialNetworkNotifier.PublishLink(post)).MustHaveHappened();
    }
}
```

Le SUT est déclaré comme une propriété avec l’attribut `[UnderTest]`. Chaque dépendance qu’on a besoin de manipuler est déclarée comme une propriété avec l’attribut `[Fake]`. La méthode `Fake.InitializeFixture` initialise le SUT, en créant les dépendances factices à la volée, et affecte ces dépendances aux propriétés correspondantes.

J’aime beaucoup le fait que les tests semblent beaucoup plus propres avec cette technique ; le code de plomberie est réduit au strict minimum, il ne reste plus qu’à configurer les dépendances et écrire les tests proprement dits.

Quelques remarques sur le code ci-dessus :

- On peut utiliser des champs privés plutôt que des propriétés publiques pour les fakes et le SUT, mais comme ces champs ne sont jamais explicitement assignés dans le code, cela provoque un avertissement du compilateur (CS0649), donc je préfère utiliser des propriétés.
- Les tests dans mon exemple utilisent [xUnit](http://xunit.github.io/), j’ai donc mis l’appel à `Fake.InitializeFixture` dans le constructeur, mais si vous utilisez un autre framework comme NUnit ou MSTest, vous pouvez le mettre dans la méthode Setup.


Notez aussi qu’il y a des limites aux scénarios supportés par cette approche :

- seule l’injection par constructeur est supportée, pas l’injection par propriétés (autrement dit les dépendances doivent être des paramètres du constructeur du SUT)
- les dépendances nommées ne sont pas supportées ; seul le type est pris en compte, on ne peut donc pas avoir plusieurs dépendances distinctes du même type
- les collections de dépendances ne sont pas supportées (c’est à dire si votre SUT reçoit une collection de dépendances, par exemple `IFooService[]`)


