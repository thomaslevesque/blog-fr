---
layout: post
title: Utiliser les informations de l'appelant de C# 5 quand on cible une version plus ancienne du .NET Framework
date: 2012-06-13T00:00:00.0000000
url: /2012/06/13/utiliser-les-informations-de-lappelant-de-c-5-quand-on-cible-une-version-plus-ancienne-du-net-framework/
tags:
  - c# 5
  - caller info
categories:
  - Astuces
---

Les [attributs d'informations de l'appelant](http://msdn.microsoft.com/fr-fr/library/hh534540%28v=vs.110%29.aspx) (*caller info attributes*) sont une des nouveautés de C# 5. Ce sont des attributs qui s'appliquent aux paramètres optionnels d'une méthode, et qui permettent de passer implicitement à cette méthode des informations sur l'appelant. Je ne suis pas sûr que cette description soit très claire, voilà donc un exemple pour bien comprendre :  
```csharp
        static void Log(
            string message,
            [CallerMemberName] string memberName = null,
            [CallerFilePath] string filePath = null,
            [CallerLineNumber] int lineNumber = 0)
        {
            Console.WriteLine(
                "[{0:g} - {1} - {2} - line {3}] {4}",
                DateTime.UtcNow,
                memberName,
                filePath,
                lineNumber,
                message);
        }
```
  La méthode ci-dessous prend plusieurs paramètres destinés à passer des informations sur l'appelant : nom du membre (méthode, propriété...) appelant, chemin du fichier source et numéro de ligne. Les attributs `Caller*` font que le compilateur va automatiquement passer les valeurs appropriées, vous n'avez donc pas besoin de les spécifier vous-même :  
```csharp
        static void Foo()
        {
            Log("Hello world");
            // Equivalent à:
            // Log("Hello world", "Foo", @"C:\x\y\z\Program.cs", 18);
        }
```
  Cela est bien sûr particulièrement utile pour les méthodes de log...  Remarquez que les attributs `Caller*` attributes sont définis dans le .NET Framework 4.5. Maintenant, supposons que l'on utilise Visual Studio 2012 pour cibler une version plus ancienne du framework (par exemple 4.0) : les attributs d'informations de l'appelant n'existent pas en 4.5, on ne peut donc pas les utiliser... Mais attendez ! Et si on pouvait faire croire au compilateur que ces attributs sont bien présents ? Définissons nos propres attributs, en prenant soin de les placer dans le namespace où le compilateur s'attend à les trouver :  
```csharp
namespace System.Runtime.CompilerServices
{
    [AttributeUsage(AttributeTargets.Parameter, AllowMultiple = false, Inherited = false)]
    public class CallerMemberNameAttribute : Attribute
    {
    }

    [AttributeUsage(AttributeTargets.Parameter, AllowMultiple = false, Inherited = false)]
    public class CallerFilePathAttribute : Attribute
    {
    }

    [AttributeUsage(AttributeTargets.Parameter, AllowMultiple = false, Inherited = false)]
    public class CallerLineNumberAttribute : Attribute
    {
    }
}
```
  Si on compile et qu'on exécute le programme, on voit que nos attributs personnalisés on bien été pris en compte par le compilateur. Il n'est donc pas nécessaire qu'ils se trouvent dans mscorlib.dll comme les "vrais" attributs d'informations de l'appelant, il suffit qu'ils soient dans le bon namespace, et le compilateur les accepte. Cela nous permet d'utiliser cette fonctionnalité quand on cible .NET 4.0, 3.5 ou même 2.0 !  Remarquez qu'une astuce similaire permet la création de méthodes d'extension quand on cible .NET 2.0 : il suffit de créer une classe `ExtensionAttribute` dans le namespace `System.Runtime.CompilerServices`. C'est d'ailleurs ce qui permet à [LinqBridge](http://www.albahari.com/nutshell/linqbridge.aspx) de fonctionner.

