---
layout: post
title: Une façon simple et sécurisée de stocker un mot de passe à l'aide de Data Protection API
date: 2013-05-26T00:00:00.0000000
url: /2013/05/26/une-facon-simple-et-securisee-de-stocker-un-mot-de-passe-a-laide-de-data-protection-api/
tags:
  - cryptographie
  - dpapi
  - mot de passe
  - sécurité
categories:
  - Code sample
---

Si vous écrivez une application cliente qui a besoin de stocker des identifiants d'utilisateur, ce n'est généralement pas une bonne idée de stocker le mot de passe en clair, pour des raisons évidentes de sécurité. Il faut donc le chiffrer, mais dès qu'on commence à envisager le chiffrement, cela soulève toutes sortes de problèmes... Quel algorithme utiliser ? Quelle clé de chiffrement ? Clairement on va avoir besoin de cette clé pour déchiffrer le mot de passe, il faut donc qu'elle se trouve dans l'exécutable ou dans la configuration. Mais dans ce cas, elle sera être assez facile à trouver...  Eh bien, la bonne nouvelle est qu'en fait vous n'avez pas vraiment besoin de résoudre ce problème, parce que Windows l'a déjà résolu pour vous ! La solution s'appelle [Data Protection API](http://msdn.microsoft.com/en-us/library/ms995355.aspx), et permet de protéger des données sans avoir à se préoccuper d'une clé de chiffrement. La documentation est un peu longue et ennuyeuse, mais en fait c'est très facile à utiliser depuis .NET, parce que le framework fournit une classe [`ProtectedData`](http://msdn.microsoft.com/en-us/library/system.security.cryptography.protecteddata.aspx) qui s'occupe d'appeler les APIs bas niveau pour vous.   Cette classe a deux méthodes, avec des noms qui se passent d'explication: `Protect` et `Unprotect`:  
```csharp
public static byte[] Protect(byte[] userData, byte[] optionalEntropy, DataProtectionScope scope);
public static byte[] Unprotect(byte[] encryptedData, byte[] optionalEntropy, DataProtectionScope scope);
```
  Le paramètre `userData` correspond aux données binaires en clair, non chiffrées. Le `scope` est une valeur qui indique si les données doivent être protégées pour l'utilisateur courant (seul cet utilisateur pourra les déchiffrer) ou pour la machine locale (n'importe quel utilisateur sur la même machine pourra les déchiffrer). Et le paramètre `optionalEntropy` ? Eh bien, je ne suis pas un expert en cryptographie, mais tel que je le comprends, c'est une sorte de "salt": d'après la documentation, il sert à "augmenter la complexité du chiffrage". Evidemment, il faudra fournir la même entropie pour déchiffrer les données plus tard. Comme son nom l'indique, ce paramètre est optionnel, il suffit de passer null si on ne veut pas l'utiliser.  Cette API est donc assez simple, mais pas directement utilisable pour notre objectif : l'entrée et la sortie sont des tableaux d'octets, mais on veux chiffrer un mot de passe, qui est une chaine de caractères; de plus, il est généralement plus pratique de stocker une chaine que des données binaires. Pour obtenir un tableau d'octets à partir d'une chaine, c'est assez simple : il suffit d'utiliser un encodage de texte, comme UTF-8. Mais on ne peut pas utiliser cette approche pour obtenir une chaine à partir des données binaires chiffrées, car elles ne correspondront sans doute pas à du texte "imprimable"; au lieu de ça, on peut encoder le résultat en Base64, qui donne une bonne représentation textuelle des données binaires. Donc, en gros voilà ce qu'on va faire :  
```
                         texte en clair
(encodage en UTF-8)   => données binaires en clair
(Protect)             => données binaires chiffrées
(encodage en base64)  => texte chiffré
```
  Et pour le déchiffrement, il suffit d'inverser les étapes:  
```
                            texte chiffré
(decodage depuis base64) => données binaires chiffrées
(Unprotect)              => données binaires en clair
(décodage depuis UTF-8)  => texte en clair
```
  Je n'ai pas parlé de l'entropie dans l'explication ci-dessus ; dans la plupart des cas il sera sans doute plus pratique de la manipuler aussi comme une chaine, donc on pourra là aussi utiliser l'encodage UTF-8 pour obtenir la forme binaire.  Finalement, on peut emballer tout ça sous forme de deux simples méthodes d'extension :  
```csharp
public static class DataProtectionExtensions
{
    public static string Protect(
        this string clearText,
        string optionalEntropy = null,
        DataProtectionScope scope = DataProtectionScope.CurrentUser)
    {
        if (clearText == null)
            throw new ArgumentNullException("clearText");
        byte[] clearBytes = Encoding.UTF8.GetBytes(clearText);
        byte[] entropyBytes = string.IsNullOrEmpty(optionalEntropy)
            ? null
            : Encoding.UTF8.GetBytes(optionalEntropy);
        byte[] encryptedBytes = ProtectedData.Protect(clearBytes, entropyBytes, scope);
        return Convert.ToBase64String(encryptedBytes);
    }
    
    public static string Unprotect(
        this string encryptedText,
        string optionalEntropy = null,
        DataProtectionScope scope = DataProtectionScope.CurrentUser)
    {
        if (encryptedText == null)
            throw new ArgumentNullException("encryptedText");
        byte[] encryptedBytes = Convert.FromBase64String(encryptedText);
        byte[] entropyBytes = string.IsNullOrEmpty(optionalEntropy)
            ? null
            : Encoding.UTF8.GetBytes(optionalEntropy);
        byte[] clearBytes = ProtectedData.Unprotect(encryptedBytes, entropyBytes, scope);
        return Encoding.UTF8.GetString(clearBytes);
    }
}
```
  Exemple de chiffrement :  
```csharp
string encryptedPassword = password.Protect();
```
  Exemple de déchiffrement :  
```csharp
try
{
    string password = encryptedPassword.Unprotect();
}
catch(CryptographicException)
{
    // Causes possible:
    // - l'entropie n'est pas celle qui a été utilisée pour le chiffrage
    // - les données ont été chiffrées par un autre utilisateur (pour scope == CurrentUser)
    // - les données ont été chiffrées sur une autre machine (pour scope == LocalMachine)
    // Dans ce cas, le mot de passe stocké n'est pas utilisable ; il faut demander à l'utilisateur de le saisir à nouveau.
}
```
  Ce que j'aime avec cette technique, c'est que ça marche tout seul : pas besoin de se préoccuper de la façon dont les données sont chiffrées, de l'endroit où est stockée la clé, ou quoi que ce soit, Windows s'occupe de tout.  Le code ci-dessus fonctionne avec le .NET Framework complet, mais la Data Protection API est également disponible : 
- pour Windows Phone : ce sont les mêmes méthodes, mais sans le paramètre scope
- pour les applications Windows Store, en utilisant la classe [DataProtectionProvider](http://msdn.microsoft.com/en-us/library/windows/apps/windows.security.cryptography.dataprotection.dataprotectionprovider). L'API est assez différente (asynchrone, pas d'entropie) et un peu plus complexe à utiliser, mais elle permet d'arriver au même résultat. [Voilà une version WinRT des méthodes d'extension ci-dessus](https://gist.github.com/thomaslevesque/5652991).


