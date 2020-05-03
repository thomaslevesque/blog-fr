---
layout: post
title: Nettoyer l'historique d'une branche Git pour supprimer les fichiers indésirables
date: 2018-03-06T00:00:00.0000000
url: /2018/03/06/nettoyer-lhistorique-dune-branche-git-pour-supprimer-les-fichiers-indesirables/
tags:
  - contrôle de version
  - git
categories:
  - Uncategorized
---


J'ai récemment eu à travailler sur un dépôt Git qui contenait des modifications à reporter sur un autre dépôt. Malheureusement, ce dépôt n'avait pas de fichier .gitignore au départ, si bien que de nombreux fichiers inutiles (répertoires bin/obj/packages...) avaient été archivés. Cela rendait l'historique très difficile à lire, puisque chaque commit contenait des centaines de fichiers modifiés.

Heureusement, Git permet assez facilement de "nettoyer" une branche, en recréant les mêmes commits sans les fichiers qui n'auraient pas dû se trouver là. Voyons donc pas-à-pas comment arriver à ce résultat.

## Mise en garde

L'opération à réaliser ici consiste en une réécriture de l'historique, une mise en garde s'impose donc : il ne faut jamais réécrire l'historique d'une branche publiée partagée avec d'autres personnes. En effet, si quelqu'un d'autre crée des commits à partir de la version actuelle de la branche, et que celle-ci est réécrite, il deviendra beaucoup plus compliqué d'intégrer ces commits à la branche réécrite.

Dans mon cas, je n'avais pas besoin de publier la branche réécrite mais seulement de l'examiner en local, donc le problème ne se posait pas. Mais n'appliquez pas cette solution à une branche sur laquelle vos collègues travaillent, si vous tenez à conserver de bonnes relations avec eux 😉.

## Créer une branche de travail

On va faire des modifications assez lourdes et potentiellement risquées sur le dépôt, il convient donc de prendre quelques précautions. Le plus simple dans ce genre de situation est tout bêtement de travailler sur une autre branche, pour ne pas risquer de faire des dégâts sur la branche originale. Par exemple, si la branche à nettoyer est `master`, on va créer une nouvelle branche `master2` à partir de `master` :

```bash
git checkout -b master2 master
```

## Identifier les fichiers à supprimer

Avant de lancer le nettoyage, il faut d'abord identifier les fichiers à supprimer. Dans le cas d'un projet .NET, il s'agit bien souvent du contenu des répertoires `bin` et `obj` (où qu'ils se trouvent) et `packages` (généralement à la racine de la solution), on va donc partir sur cette hypothèse pour l'instant. Les patterns des fichiers à supprimer sont donc les suivants :

- `**/bin/**`
- `**/obj/**`
- `packages/**`


## Nettoyer la branche : la commande git filter-branch

La commande Git qui va nous permettre de supprimer les fichiers indésirables s'appelle [`filter-branch`](https://git-scm.com/docs/git-filter-branch). Elle est décrite dans le livre [Pro Git](https://git-scm.com/book/fr/v2/Utilitaires-Git-R%C3%A9%C3%A9crire-l%E2%80%99historique#_l_option_nucl%C3%A9aire_code_filter_branch_code) comme "l'option nucléaire", car elle est très puissante et potentiellement dévastatrice... elle est donc à manipuler avec précaution.

Le principe de cette commande est de reprendre chaque commit de la branche, lui appliquer un filtre, et le recommiter avec les modifications causées par le filtre. Il existe plusieurs types de filtre, par exemple :

- `--msg-filter` : permet de réécrire les messages des commits de la branche.
- `--tree-filter` : permet de filtrer les fichiers au niveau de la copie de travail du dépôt (effectue un checkout de chaque commit, ce qui peut être assez long sur un gros dépôt)
- `--index-filter` : permet de filtrer les fichiers au niveau de l'index (ne nécessite pas un checkout de chaque commit, donc plus rapide).


Dans notre scénario, `--index-filter` est parfaitement indiqué, vu qu'on souhaite simplement filtrer les fichiers par rapport à leur chemin. La commande `filter-branch` avec ce type de filtre s'utilise comme ceci :

```bash
git filter-branch --index-filter '<command>'
```

`<command>` désigne une commande bash qui sera exécutée pour chaque commit de la branche à réécrire. Dans le cas qui nous intéresse, ce sera simplement un appel à `git rm` pour supprimer de l'index les fichiers indésirables :

```bash
git filter-branch --index-filter 'git rm --cached --ignore-unmatch **/bin/** **/obj/** packages/**'
```

Le paramètre `--cached` indique qu'on travaille sur l'index et non sur la copie de travail; `--ignore-unmatch` permet d'ignorer les cas où aucun fichier ne correspond au pattern spécifié. Par défaut, la commande s'applique uniquement à la branche courante.

Pour peu que la branche ait beaucoup d'historique, la commande peut prendre assez longtemps à s'exécuter, il faudra donc s'armer de patience... Une fois terminé, vous devriez avoir une branche contenant des commits identiques à ceux de la branche d'origine, mais sans les fichiers indésirables.

## Cas plus complexes

Dans l'exemple ci-dessus, il n'y avait que 3 patterns de fichiers à supprimer, donc la commande était assez courte pour être écrite "inline". Mais s'il y en a beaucoup plus, ou si la logique à appliquer pour supprimer les fichiers est plus complexe, ça ne tient plus vraiment la route... le plus simple est donc d'écrire un script (bash) qui contient toutes les commandes nécessaires, et de passer ce script en paramètre de `git filter-branch --index-filter`.

## Nettoyer uniquement à partir d'un commit spécifique

Dans l'exemple précédent, on applique `filter-branch` à la totalité de la branche. Mais il est également possible de ne l'appliquer qu'à partir d'un commit spécifique, en spécifiant une plage de commits :

```bash
git filter-branch --index-filter '<command>' <ref>..HEAD
```

Ici, `<ref>` désigne une référence de commit (SHA1, branche ou tag). Notez que la fin de la plage de commits doit forcément être `HEAD` : on ne peut pas réécrire le début ou le milieu d'une branche sans toucher aux commits suivants, puisque le SHA1 de chaque commit dépend du commit précédent.

