---
layout: post
title: '[WPF] Une grille simplifiée utilisant des attributs XAML'
date: 2010-07-20T00:00:00.0000000
url: /2010/07/20/wpf-une-grille-simplifiee-utilisant-des-attributs-xaml/
tags:
  - grid
  - WPF
  - XAML
categories:
  - Code sample
  - WPF
---

Le composant `Grid` est l'un des contrôles les plus utilisés en WPF. Il permet de disposer facilement des éléments selon des lignes et des colonnes. Malheureusement le code pour l'utiliser, bien que simple à écrire, est relativement lourd :  
```xml
<Grid>
    <Grid.RowDefinitions>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="5"/>
        <RowDefinition Height="*"/>
    </Grid.RowDefinitions>
    <Grid.ColumnDefinitions>
        <ColumnDefinition Width="60" />
        <ColumnDefinition Width="*" />
    </Grid.ColumnDefinitions>
    
    <Label Content="Name" Grid.Row="0" Grid.Column="0" />
    <TextBox Text="Hello world" Grid.Row="0" Grid.Column="1"/>
    <Rectangle Fill="Black" Grid.Row="1" Grid.ColumnSpan="2"/>
    <Label Content="Image" Grid.Row="2" Grid.Column="0" />
    <Image Source="Resources/Desert.jpg" Grid.Row="2" Grid.Column="1" />
</Grid>
```
  Dans cet exemple, plus de la moitié du code est constitué de la définition de la grille ! Bien que cette syntaxe offre une certaine souplesse et permette de contrôler assez finement la disposition, dans la plupart des cas on a seulement besoin de définir la hauteur des lignes ou la largeur des colonnes... il serait donc beaucoup plus simple de pouvoir déclarer la grille de cette façon :  
```xml
<Grid Rows="Auto,5,*" Columns="60,*">
    ...
</Grid>
```
  La suite de cet article démontre comment atteindre précisément ce résultat, en créant une classe `SimpleGrid` héritée de `Grid`.  Pour commencer, notre classe aura besoin de deux nouvelles propriétés : `Rows` et `Columns`. Ces propriétés définissent respectivement les hauteurs et largeurs des lignes et des colonnes. Ces dimensions ne sont pas de simples nombres : en effet, des valeurs comme `"*"`, `"2*"` ou `"Auto"` sont des dimensions valides. Il existe en WPF un type dédié pour représenter ces dimensions : la structure `GridLength`. Nos deux propriétés seront donc des collections de `GridLength`. Voilà donc la signature de la classe `SimpleGrid` :  
```csharp
public class SimpleGrid : Grid
{
    public IList<GridLength> Rows { get; set; }
    public IList<GridLength> Columns { get; set; }
}
```
  Puisque ce sont ces propriétés qui vont contrôler les lignes et colonnes de la grille, il faut qu'elles modifient les `RowDefinitions` et `ColumnDefinitions` de la classe de base. Voilà donc comment les implémenter pour obtenir le résultat voulu : 
```csharp
        private IList<GridLength> _rows;
        public IList<GridLength> Rows
        {
            get { return _rows; }
            set
            {
                _rows = value;
                RowDefinitions.Clear();
                if (_rows == null)
                    return;
                foreach (var length in _rows)
                {
                    RowDefinitions.Add(new RowDefinition { Height = length });
                }
            }
        }

        private IList<GridLength> _columns;
        public IList<GridLength> Columns
        {
            get { return _columns; }
            set
            {
                _columns = value;
                ColumnDefinitions.Clear();
                if (_columns == null)
                    return;
                foreach (var length in _columns)
                {
                    ColumnDefinitions.Add(new ColumnDefinition { Width = length });
                }
            }
        }
```
  Notre classe `SimpleGrid` est d'ores et déjà utilisable... à partir du code C#, ce qui ne nous aide pas beaucoup quand il s'agit de simplifier le code XAML. Il nous faut donc trouver un moyen de déclarer dans un attribut les valeurs de ces propriétés, ce qui n'est pas évident dans la mesure où ce sont des collections...  En XAML, tous les attributs sont écrits sous forme de chaines de caractères. Pour convertir ces chaines en valeurs du type voulu, WPF fait appel à des convertisseurs, qui sont des classes héritées de `TypeConverter`, associées à chaque type qui supporte les conversions de et vers d'autres types. Par exemple, le type `GridLength` a pour convertisseur le type `GridLengthConverter`, qui permet de convertir des nombres ou des chaines de caractères en `GridLength`, et inversement. Le mécanisme de conversion est décrit dans [cet article](http://msdn.microsoft.com/fr-fr/library/aa970913.aspx) sur MSDN.  Nous allons donc devoir créer un convertisseur et l'associer au type de nos propriétés. Comme nous n'avons pas la main sur le type `IList<T>`, nous allons d'abord créer un type spécifique `GridLengthCollection` qu'on utilisera à la place de `IList<GridLength>`, et lui associer un convertisseur `GridLengthCollectionConverter` :  
```csharp
    [TypeConverter(typeof(GridLengthCollectionConverter))]
    public class GridLengthCollection : ReadOnlyCollection<GridLength>
    {
        public GridLengthCollection(IList<GridLength> lengths)
            : base(lengths)
        {
        }
    }
```
  Pourquoi une collection en lecture seule ? Tout bêtement parce que permettre d'ajouter ou de supprimer des lignes ou des colonnes compliquerait l'implémentation, et n'apporterait rien pour l'objectif qui nous intéresse, à savoir simplifier la déclaration de la grille en XAML. Donc, restons dans la simplicité... Pour éviter de réinventer la roue, on hérite de la classe `ReadOnlyCollection<T>`, qui correspond parfaitement à notre besoin.  Notez aussi l'utilisation de l'attribut `TypeConverter` : il sert à indiquer le convertisseur associé à un type. Il nous reste donc simplement à implémenter ce convertisseur :  
```csharp
    public class GridLengthCollectionConverter : TypeConverter
    {
        public override bool CanConvertFrom(ITypeDescriptorContext context, Type sourceType)
        {
            if (sourceType == typeof(string))
                return true;
            return base.CanConvertFrom(context, sourceType);
        }

        public override bool CanConvertTo(ITypeDescriptorContext context, Type destinationType)
        {
            if (destinationType == typeof(string))
                return true;
            return base.CanConvertTo(context, destinationType);
        }

        public override object ConvertFrom(ITypeDescriptorContext context, System.Globalization.CultureInfo culture, object value)
        {
            string s = value as string;
            if (s != null)
                return ParseString(s, culture);
            return base.ConvertFrom(context, culture, value);
        }

        public override object ConvertTo(ITypeDescriptorContext context, CultureInfo culture, object value, Type destinationType)
        {
            if (destinationType == typeof(string) && value is GridLengthCollection)
                return ToString((GridLengthCollection)value, culture);
            return base.ConvertTo(context, culture, value, destinationType);
        }

        private string ToString(GridLengthCollection value, CultureInfo culture)
        {
            var converter = new GridLengthConverter();
            return string.Join(",", value.Select(v => converter.ConvertToString(v)));
        }

        private GridLengthCollection ParseString(string s, CultureInfo culture)
        {
            var converter = new GridLengthConverter();
            var lengths = s.Split(',').Select(p => (GridLength)converter.ConvertFromString(p.Trim()));
            return new GridLengthCollection(lengths.ToArray());
        }
    }
```
  Ce convertisseur est capable de convertir une `GridLengthCollection` de et vers une chaine de caractères, dans laquelle les dimensions sont séparées par des virgules. Notez l'utilisation du convertisseur `GridLengthConverter` : puisqu'il existe déjà un convertisseur pour les éléments de notre collection, autant s'en servir...  Toutes les pièces du puzzle étant en place, il ne nous reste plus qu'à utiliser notre nouvelle grille simplifiée :  
```xml
<my:SimpleGrid Rows="Auto,5,*" Columns="60,*">
    <Label Content="Name" Grid.Row="0" Grid.Column="0" />
    <TextBox Text="Hello world" Grid.Row="0" Grid.Column="1"/>
    <Rectangle Fill="Black" Grid.Row="1" Grid.ColumnSpan="2"/>
    <Label Content="Image" Grid.Row="2" Grid.Column="0" />
    <Image Source="Resources/Desert.jpg" Grid.Row="2" Grid.Column="1" />
</my:SimpleGrid>
```
  On obtient donc un résultat beaucoup plus concis et lisible qu'en utilisant une `Grid` normale, l'objectif est donc atteint :)  On pourrait bien sûr envisager des améliorations, par exemple déclarer les propriétés `Rows` et `Columns` comme des `DependencyProperty` afin de permettre le binding, ou encore gérer l'ajout et la suppression de colonnes. Cependant, cette grille s'adresse à des scénarios simples où la grille est définie une fois pour toutes et n'est pas modifiée à l'exécution (ce qui correspond a priori au cas d'utilisation le plus fréquent), il semble donc plus judicieux de la garder la plus simple possible. Pour des besoins plus spécifiques, par exemple si l'on veut utiliser les propriétés `MinWidth`, `MaxWidth` ou encore `SharedSizeGroup`, il faudra donc revenir à la `Grid` standard.  Pour référence, voici le code final de la classe `SimpleGrid` :  
```csharp
    public class SimpleGrid : Grid
    {
        private GridLengthCollection _rows;
        public GridLengthCollection Rows
        {
            get { return _rows; }
            set
            {
                _rows = value;
                RowDefinitions.Clear();
                if (_rows == null)
                    return;
                foreach (var length in _rows)
                {
                    RowDefinitions.Add(new RowDefinition { Height = length });
                }
            }
        }

        private GridLengthCollection _columns;
        public GridLengthCollection Columns
        {
            get { return _columns; }
            set
            {
                _columns = value;
                if (_columns == null)
                    return;
                ColumnDefinitions.Clear();
                foreach (var length in _columns)
                {
                    ColumnDefinitions.Add(new ColumnDefinition { Width = length });
                }
            }
        }
    }
```

