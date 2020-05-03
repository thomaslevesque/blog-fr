---
layout: post
title: '[WPF] Article de Josh Smith sur le design pattern Model-View-ViewModel'
date: 2009-02-25T00:00:00.0000000
url: /2009/02/25/wpf-article-de-josh-smith-sur-le-design-pattern-model-view-viewmodel/
tags:
  - article
  - design pattern
  - Josh Smith
  - MVVM
  - WPF
categories:
  - WPF
---

Depuis l'apparition de WPF, on entend de plus en plus souvent parler de "Model-View-ViewModel" (MVVM). Il s'agit d'un design pattern, inspiré entre autres de Model-View-Controller (MVC) et Presentation Model (PM), conçu spécifiquement pour tirer le meilleur parti des fonctionnalités de WPF. Ce pattern permet notamment un excellent découplage entre les données, le comportement, et la présentation des données, ce qui rend le code plus facile à comprendre et à maintenir, et facilite la collaboration entre un développeur et un designer. Un autre avantage de MVVM est qu'il est permet d'écrire des programmes facilement testables.  Pour tout savoir sur le pattern MVVM, je vous invite à lire l'excellent article de Josh Smith à ce sujet, paru dans l'édition de février du MSDN Magazine : [WPF Apps With The Model-View-ViewModel Design Pattern](http://msdn.microsoft.com/en-us/magazine/dd419663.aspx) (en anglais).  A travers un exemple simple mais concret, Josh Smith aborde pratiquement tous les aspects du pattern MVVM, notamment : 
- Data binding
- Commandes
- Validation
- Tests unitaires
- ...

  Le code source fourni constitue de plus une bonne base de départ pour une application développée selon le pattern MVVM, ainsi qu'une mine d'exemples concrets.

