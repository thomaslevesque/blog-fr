---
layout: post
title: Envoyer des données avec HttpClient selon un modèle “push”
date: 2013-12-01T00:00:00.0000000
url: /2013/12/01/envoyer-des-donnees-avec-httpclient-selon-un-modele-push/
tags:
  - HttpClient
  - HttpContent
  - JSON
  - ObjectContent
  - push
  - PushStreamContent
categories:
  - Uncategorized
---


Si vous avez déjà utilisé la classe `HttpWebRequest` pour envoyer des données, vous savez qu’elle utilise un modèle “push”. Ce que j’entends par là, c’est que vous appelez la méthode `GetRequestStream`, qui ouvre la connexion si nécessaire, envoie les en-têtes, et renvoie un flux sur lequel vous pouvez écrire directement.

.NET 4.5 a introduit la classe `HttpClient` comme nouveau moyen de communiquer en HTTP. Elle repose en fait sur `HttpWebRequest` en interne, mais offre une API plus pratique et complètement asynchrone. `HttpClient` utilise une approche différente en ce qui concerne l’upload de données : au lieu d’écrire directement sur le flux de la requête, il faut affecter à la propriété `Content` du `HttpRequestMessage` une instance d’une classe dérivée de `HttpContent`. On peut également passer le contenu directement aux méthodes `PostAsync` ou `PutAsync`.

Le framework .NET fournit quelques implémentations standard de `HttpContent`, voici quelques unes des plus communément utilisées :

- `ByteArrayContent`: représente du contenu binaire brut en mémoire
- `StringContent`: représente du texte avec un encodage spécifique (c’est une spécialisation de `ByteArrayContent`)
- `StreamContent`: représente des données binaires brutes sous forme d’un flux


Par exemple, voilà comment on peut uploader le contenu d’un fichier :

```
async Task UploadFileAsync(Uri uri, string filename)
{
    using (var stream = File.OpenRead(filename))
    {
        var client = new HttpClient();
        var response = await client.PostAsync(uri, new StreamContent(stream));
        response.EnsureSuccessStatusCode();
    }
}
```

Comme vous l’aurez peut-être remarqué, ce code n’écrit jamais explicitement sur le flux de la requête : le contenu est automatiquement lu depuis le flux source et copié vers le flux de la requête.

Ce modèle “pull” convient à la plupart des usages, mais il a un inconvénient : il nécessite que les données à envoyer existent déjà sous une forme qui peut être directement envoyée au serveur. Ce n’est pas toujours souhaitable, parce que parfois on veut générer “à la volée” le contenu de la requête. Par exemple, si on veut envoyer un objet sérialisé en JSON, avec l’approche “pull”, il faut d’abord le sérialiser en mémoire dans une `String` ou un `MemoryStream`, puis affecter cela au contenu de la requête :

```
async Task UploadJsonObjectAsync<T>(Uri uri, T data)
{
    var client = new HttpClient();
    string json = JsonConvert.SerializeObject(data);
    var response = await client.PostAsync(uri, new StringContent(json));
    response.EnsureSuccessStatusCode();
}
```

Ce n’est pas vraiment un problème pour des petits objets, mais ce n’est clairement pas optimal pour des graphes d’objets plus importants…

Alors, comment pourrait-on inverser ce modèle pull en un modèle push ? Eh bien c’est en fait assez simple : il suffit de créer une classe qui hérite de `HttpContent`, et de redéfinir sa méthode `SerializeToStreamAsync` pour écrire directement sur le flux de la requête. En fait, j’avais l’intention de bloguer sur ma propre implémentation, mais j’ai fait quelques recherches, et il s’avère que Microsoft a déjà fait le travail : la librairie [Web API 2 Client](https://www.nuget.org/packages/Microsoft.AspNet.WebApi.Client) fournit une classe `PushStreamContent` qui fait exactement ça. En gros, il faut juste lui passer un délégué qui définit quoi faire avec le flux de la requête. Voilà comment ça fonctionne :

```
async Task UploadJsonObjectAsync<T>(Uri uri, T data)
{
    var client = new HttpClient();
    var content = new PushStreamContent((stream, httpContent, transportContext) =>
    {
        var serializer = new JsonSerializer();
        using (var writer = new StreamWriter(stream))
        {
            serializer.Serialize(writer, data);
        }
    });
    var response = await client.PostAsync(uri, content);
    response.EnsureSuccessStatusCode();
}
```

Notez que la classe `PushStreamContent` fournit aussi une surcharge du constructeur qui accepte un délégué asynchrone, si vous voulez écrire sur le flux de façon asynchrone.

En fait, pour ce cas d’utilisation spécifique, la librairie Web API 2 Client propose une approche moins alambiquée : la classe `ObjectContent`. Il faut juste lui passer l’objet à envoyer ainsi qu’un `MediaTypeFormatter`, et elle se charge de sérialiser l’objet sur le flux de la requête :

```
async Task UploadJsonObjectAsync<T>(Uri uri, T data)
{
    var client = new HttpClient();
    var content = new ObjectContent<T>(data, new JsonMediaTypeFormatter());
    var response = await client.PostAsync(uri, content);
    response.EnsureSuccessStatusCode();
}
```

Par défaut, la classe `JsonMediaTypeFormatter` utilise [Json.NET](http://james.newtonking.com/json) comme sérialiseur JSON, mais il y a une option pour utiliser `DataContractJsonSerializer` à la place.

Notez que si vous voulez lire un objet depuis le contenu de la réponse, c’est encore plus facile : utilisez simplement la méthode d’extension [ReadAsAsync&lt;T&gt;](http://msdn.microsoft.com/en-us/library/system.net.http.httpcontentextensions.readasasync%28v=vs.118%29.aspx) (également dans la librairie Web API 2 Client). Comme vous pouvez le voir, il est donc extrêmement facile de consommer des APIs REST à l’aide de `HttpClient`.

