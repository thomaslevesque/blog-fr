---
layout: post
title: '[WPF] Binding asynchrone sur une propriété du ViewModel'
date: 2009-04-01T00:00:00.0000000
url: /2009/04/01/wpf-binding-asynchrone-sur-une-propriete-du-viewmodel/
tags:
  - asynchrone
  - binding
  - markup extension
  - MVVM
  - WPF
categories:
  - Code sample
  - WPF
---

***Mise à jour** : Comme l'a très justement indiqué Jérémy en commentaire, la propriété IsAsync du Binding permet de faire à peu près la même chose beaucoup plus simplement... Bien que ma méthode puisse servir pour certains besoins spécifiques, dans la plupart des cas la propriété IsAsync est probablement le meilleur choix ! Je laisse le billet malgré tout, ne serait-ce que pour la classe SwitchBinding qui me semble assez utile...*  J'ai eu récemment besoin, dans une application basée sur le pattern MVVM, d'afficher une propriété dont la valeur était assez longue à obtenir (récupérer par une requête HTTP). Au départ, j'ai simplement implémenté la propriété en suivant le principe du *lazy loading* : le binding sur cette propriété provoquait donc l'obtention de la valeur par une requête HTTP. Le résultat, prévisible, était un gel de l'interface pendant la récupération de la valeur. La solution classique pour ce genre de problème est de récupérer la valeur sur un autre thread, et d'affecter le résultat au contrôle qui doit l'afficher... sauf qu'en MVVM on n'a pas accès à ce contrôle. Une autre approche, plus adaptée, est d'affecter la valeur à la propriété du ViewModel, ce qui déclenche l'évènement `PropertyChanged` et rafraichit la vue.  J'ai essayé pas mal de choses avant d'arriver à une solution avec le moins possible de code de "plomberie", je vais donc vous la faire partager. Voilà le code de la propriété :  
```csharp
        private bool _retrievingValue = false;

        private object _value;
        public object Value
        {
            get
            {
                if (_value == null && !_retrievingValue)
                {
                    _retrievingValue = true;
                    ThreadPool.QueueUserWorkItem(
                        (state) =>
                        {
                            this.Value = _model.RetrieveValue(); // Very long operation...
                            _retrievingValue = false;
                        });
                }
                return _value;
            }
            set
            {
                _value = value;
                OnPropertyChanged("Value");
            }
        }
```
  Ce code est assez simple à comprendre, mais mérite quand même quelques commentaires : 
- Le premier binding sur cette propriété récupère d'abord une valeur nulle, mais déclenche aussi la récupération asynchrone de la valeur. Notez le flag `_retrievingValue` qui évite de lancer plusieurs fois la récupération
- Quand la récupération de la valeur est terminée, la propriété est mise à jour, et l'évènement `PropertyChanged` met à jour le binding
- Un détail intéressant est que la propriété est mise à jour directement dans le thread de travail. Puisque cette mise à jour provoque une modification de la vue, on aurait pu s'attendre à une `InvalidOperationException`, car on ne peut pas modifier la vue à partir d'un autre thread... mais en fait, le mécanisme de binding de WPF est lui-même asynchrone, ce qui masque la complexité de l'appel cross-thread. Il est donc inutile de recourir à un Dispatcher.Invoke ou autre pirouette, ce qui simplifie bien la vie des développeurs que nous sommes...
**Attention :**ce système de binding asynchrone fonctionne bien pour affecter une valeur à une propriété du ViewModel, mais ne permet pas, par exemple, de modifier les éléments d'une ObservableCollection. Si vous essayez, à partir d'un autre thread, d'ajouter ou enlever des éléments à une collection sur laquelle la vue est bindée, cela provoquera une`NotSupportedException`:

> Ce type de CollectionView ne prend pas en charge les modifications de son SourceCollection à partir d’un thread différent du thread du Dispatcher.

Pour modifier des collections de façon asynchrone, il faudra donc se débrouiller autrement... cela fera l'objet d'un prochain billet si je trouve une solution satisfaisante (peut-être à base d'`AsyncOperation`...). Si vous avez une idée là-dessus, n'hésitez pas à la poster en commentaire !<br><br>Fermons cette parenthèse et revenons à notre propriété Value. La méthode décrite plus haut fonctionne bien et nécessite assez peu de code, mais elle a un inconvénient : pendant la récupération de la valeur, l'utilisateur ne voit rien... On aimerait pouvoir afficher quelque chose qui indique que l'application travaille. Pour ça, on peut introduire une propriété`IsValueReady`qui indiquera si la valeur est prête. Côté XAML, on pourra utiliser un Trigger sur cette propriété pour modifier l'affichage.
```csharp
        private bool _retrievingValue = false;

        private object _value;
        public object Value
        {
            get
            {
                if (!_isValueReady && !_retrievingValue)
                {
                    _retrievingValue = true;
                    ThreadPool.QueueUserWorkItem(
                        (state) =>
                        {
                            this.Value = _model.RetrieveValue(); // Very long operation...
                            this.IsValueReady = true;
                            _retrievingValue = false;
                        });
                }
                return _value;
            }
            set
            {
                _value = value;
                OnPropertyChanged("Value");
            }
        }

        private bool _isValueReady = false;
        public bool IsValueReady
        {
            get { return _isValueReady; }
            private set
            {
                _isValueReady = value;
                OnPropertyChanged("IsValueReady");
            }
        }
```
Ça commence à faire un code un peu plus conséquent pour une simple propriété, mais ce code est toujours le même... les seules choses qui changent sont le nom de la propriété, son type, et le code qui récupère la valeur. Si on a beaucoup de propriétés de ce genre à créer, on pourrait donc facilement écrire un*code snippet*qui génèrerait le plus gros du code.<br><br>Avec Blend, il est probablement assez simple de créer un trigger pour prendre en compte la propriété IsValueReady dans la vue... mais je ne me suis toujours pas mis à Blend, je code directement la vue en XAML, et je trouve les Triggers beaucoup trop lourds à écrire... J'ai donc utilisé une autre solution que je trouve  beaucoup plus simple et plus lisible, à base de markup extension (oui, j'aime bien les markup extensions...). Ca donne le XAML suivant :
```xml
    <Grid>
        <TextBlock Text="{Binding Value, FallbackValue=Blabla}"
                   Visibility="{my:SwitchBinding IsValueReady, Visible, Hidden}"/>
        <ProgressBar IsIndeterminate="True" Width="150" Height="30"
                     Visibility="{my:SwitchBinding IsValueReady, Hidden, Visible}"/>
    </Grid>
```
Ce code masque le`TextBlock`et affiche la`ProgressBar`tant que la valeur n'est pas prête. Quand la récupération de la valeur est terminée, la`ProgressBar`disparait et le`TextBlock`redevient visible...`SwitchBinding`est une markup extension qui hérite de`Binding`et renvoie une valeur ou une autre selon que la propriété bindée vaut`true`ou`false`. Je ne m'étendrai pas sur le fonctionnement de cette extension, car ce n'est pas le sujet de ce billet, mais voici tout de même son code :
```csharp
    public class SwitchBindingExtension : Binding
    {
        public SwitchBindingExtension()
        {
            Initialize();
        }

        public SwitchBindingExtension(string path)
            : base(path)
        {
            Initialize();
        }

        public SwitchBindingExtension(string path, object valueIfTrue, object valueIfFalse)
            : base(path)
        {
            Initialize();
            this.ValueIfTrue = valueIfTrue;
            this.ValueIfFalse = valueIfFalse;
        }

        private void Initialize()
        {
            this.ValueIfTrue = Binding.DoNothing;
            this.ValueIfFalse = Binding.DoNothing;
            this.Converter = new SwitchConverter(this);
        }

        [ConstructorArgument("valueIfTrue")]
        public object ValueIfTrue { get; set; }

        [ConstructorArgument("valueIfFalse")]
        public object ValueIfFalse { get; set; }

        private class SwitchConverter : IValueConverter
        {
            public SwitchConverter(SwitchBindingExtension switchExtension)
            {
                _switch = switchExtension;
            }

            private SwitchBindingExtension _switch;

            #region IValueConverter Members

            public object Convert(object value, Type targetType, object parameter, System.Globalization.CultureInfo culture)
            {
                try
                {
                    bool b = System.Convert.ToBoolean(value);
                    return b ? _switch.ValueIfTrue : _switch.ValueIfFalse;
                }
                catch
                {
                    return DependencyProperty.UnsetValue;
                }
            }

            public object ConvertBack(object value, Type targetType, object parameter, System.Globalization.CultureInfo culture)
            {
                return Binding.DoNothing;
            }

            #endregion
        }

    }
```
Cette markup extension est en fait l'équivalent XAML de l'opérateur conditionnel de C# (`condition ? valeurSiVrai : valeurSiFaux`), et peut être utilisée pour toutes sortes de valeurs.<br><br>Une autre option pour réaliser le comportement voulu aurait été de créer des propriétés qui renvoient une valeur de Visibility selon la valeur de IsValueReady, mais ça fait encore 2 propriétés de plus à créer, ce qui alourdit pas mal le ViewModel.<br><br>Voilà, c'est tout pour aujourd'hui... N'hésitez pas à me faire part de vos commentaires ou suggestions ;)

