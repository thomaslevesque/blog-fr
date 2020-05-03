---
layout: post
title: Passage de paramètres par référence à une méthode asynchrone
date: 2014-11-04T00:00:00.0000000
url: /2014/11/04/passing-parameters-by-reference-to-an-asynchronous-method/
tags:
  - async
  - asynchrone
  - await
  - byref
  - C#
  - out
  - ref
  - reference
categories:
  - Astuces
---


L’asynchronisme dans C# est une fonctionnalité géniale, et je l’ai beaucoup utilisé depuis son apparition. Mais il y a quelques limitations agaçantes; par exemple, on ne peut pas passer des paramètres par référence (`ref` ou `out`) à une méthode asynchrone. Il y a de bonnes raisons pour cela; la plus évidente est que si vous passez par référence une variable locale, elle est stockée sur la pile, or la pile ne va pas rester disponible pendant toute l’exécution de la méthode asynchone (seulement jusqu’au premier `await`), donc l’emplacement de la variable n’existera plus.

Cependant, cette limitation est assez facile à contourner : il suffit de créer une classe `Ref<T>` pour encapsuler la valeur, et de passer une instance de cette classe par valeur à la méthode asynchrone:

```
async void btnFilesStats_Click(object sender, EventArgs e)
{
    var count = new Ref<int>();
    var size = new Ref<ulong>();
    await GetFileStats(tbPath.Text, count, size);
    txtFileStats.Text = string.Format("{0} files ({1} bytes)", count, size);
}

async Task GetFileStats(string path, Ref<int> totalCount, Ref<ulong> totalSize)
{
    var folder = await StorageFolder.GetFolderFromPathAsync(path);
    foreach (var f in await folder.GetFilesAsync())
    {
        totalCount.Value += 1;
        var props = await f.GetBasicPropertiesAsync();
        totalSize.Value += props.Size;
    }
    foreach (var f in await folder.GetFoldersAsync())
    {
        await GetFilesCountAndSize(f, totalCount, totalSize);
    }
}
```

La class `Ref<T>` ressemble à ceci:

```
public class Ref<T>
{
    public Ref() { }
    public Ref(T value) { Value = value; }
    public T Value { get; set; }
    public override string ToString()
    {
        T value = Value;
        return value == null ? "" : value.ToString();
    }
    public static implicit operator T(Ref<T> r) { return r.Value; }
    public static implicit operator Ref<T>(T value) { return new Ref<T>(value); }
}
```

Comme vous pouvez le voir, il n’y a rien de très compliqué. Cette approche peut également être utilisée pour les blocs itérateurs (`yield return`), qui n’autorisent pas non plus les paramètres `ref` ou `out`. Elle a aussi un avantage par rapport aux paramètres `ref` et `out` standards: elle permet de rendre le paramètre optionel, par exemple si on n’est pas intéressé par le résultat (évidemment il faut que la méthode appelée gère ce cas de façon appropriée).

