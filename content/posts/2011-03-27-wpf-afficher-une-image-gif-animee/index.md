---
layout: post
title: '[WPF] Afficher une image GIF animée'
date: 2011-03-27T00:00:00.0000000
url: /2011/03/27/wpf-afficher-une-image-gif-animee/
tags:
  - animé
  - gif
  - WPF
categories:
  - WPF
---

WPF est une technologie géniale, mais parfois on a l'impression qu'il lui manque certaines fonctionnalités assez basiques... Un exemple souvent cité est l'absence de support pour les images GIF animées. En fait, le format GIF proprement dit est supporté, mais le contrôle `Image` n'affiche que la première image de l'animation.  De nombreuses solutions à ce problème ont été proposées sur les forums et blogs techniques, généralement des variantes autour des approches suivantes :  
- Utiliser le contrôle `MediaElement` : malheureusement ce contrôle ne supporte que les URI de type `file://` ou `http://`, et non le schéma d'URI `pack://` utilisé pour les ressources WPF ; l'image ne peut donc pas être inclue dans les ressources, elle doit être dans un fichier à part. De plus, la transparence n'est pas supportée, si bien que le résultat final est assez laid
- Utiliser le contrôle `PictureBox` de Windows Forms, via un `WindowsFormsHost` : personnellement j'ai horreur d'utiliser des contrôles Windows Forms en WPF, ça me donne l'impression de faire quelque chose de mal :P
- Créer un contrôle dérivé de `Image` qui gère l'animation. Pour l'implémentation, certaines solutions tirent partie de la classe `ImageAnimator` de `System.Drawing` (GDI), d'autres utilisent une animation WPF pour changer de frame. C'est une approche assez "propre", mais qui oblige à utiliser un contrôle spécifique pour les GIF. De plus la solution utilisant `ImageAnimator` se révèle assez peu fluide.

  Comme vous l'aurez deviné, aucune de ces solutions ne me satisfait vraiment... De plus, aucune ne gère proprement la durée de chaque frame, et suppose simplement que toutes les frames durent 100ms (c'est presque toujours le cas, mais le *presque*fait toute la différence...). Je n'ai donc gardé que les meilleures idées dans les approches ci-dessus pour créer ma propre solution. Les objectifs que je souhaitais atteindre sont les suivants :
- Ne pas dépendre de Windows Forms ou de GDI
- Afficher l'image animée dans un contrôle `Image` standard
- Pouvoir utiliser le même code XAML pour une image fixe ou animée
- Supporter la transparence
- Tenir compte de la durée réelle de chaque frame de l'image

  Pour arriver à ce résultat, je suis parti d'une idée simple, voire évidente : pour animer l'image, il suffit d'appliquer une animation à la propriété `Source` du contrôle `Image`. Or WPF fournit tous les outils nécessaires pour réaliser ce type d'animation ; en l'occurrence la classe [`ObjectAnimationUsingKeyFrames`](http://msdn.microsoft.com/fr-fr/library/system.windows.media.animation.objectanimationusingkeyframes.aspx) répond parfaitement au besoin : on peut spécifier à quel instant exact affecter une valeur donnée à la propriété, ce qui permet de tenir compte de la durée des frames.  Le problème suivant est d'extraire les différentes frames de l'image : heureusement ce scénario est prévu dans WPF, et la classe [`BitmapDecoder`](http://msdn.microsoft.com/fr-fr/library/system.windows.media.imaging.bitmapdecoder.aspx) fournit une propriété `Frames` qui sert à ça. Donc, pas de difficulté majeure à ce niveau...  Enfin, dernier obstacle : extraire la durée de chaque frame. C'est finalement la partie qui m'a demandé le plus de recherche... J'ai d'abord cru qu'il faudrait lire manuellement le fichier pour trouver cette information, en décodant directement les données binaires. Mais la solution est finalement assez simple, et tire partie de la classe [`BitmapMetadata`](http://msdn.microsoft.com/fr-fr/library/system.windows.media.imaging.bitmapmetadata.aspx). La seule difficulté a été de localiser le "chemin" de la métadonnée qui contient cette information, mais après quelques tâtonnements, la voilà : `/grctlext/Delay`.  La solution finale est implémentée sous forme d'une propriété attachée `AnimatedSource` applicable au contrôle `Image`, qui s'utilise en lieu et place de `Source` :  
```xml
<Image Stretch="None" my:ImageBehavior.AnimatedSource="/Images/animation.gif" />
```
  On peut également affecter une image fixe à cette propriété, elle s'affichera normalement ; on peut donc utiliser cette propriété sans se soucier de savoir si l'image à afficher sera fixe ou animée.  Au final, tous les objectifs fixés au départ sont donc atteints, et il y a même une cerise sur le gâteau : cette solution fonctionne également dans le designer (du moins dans Visual Studio 2010), on voit donc directement l'animation quand on affecte la propriété `AnimatedSource` :)  Sans plus attendre, voilà le code complet :  
```csharp
    public static class ImageBehavior
    {
        #region AnimatedSource

        [AttachedPropertyBrowsableForType(typeof(Image))]
        public static ImageSource GetAnimatedSource(Image obj)
        {
            return (ImageSource)obj.GetValue(AnimatedSourceProperty);
        }

        public static void SetAnimatedSource(Image obj, ImageSource value)
        {
            obj.SetValue(AnimatedSourceProperty, value);
        }

        public static readonly DependencyProperty AnimatedSourceProperty =
            DependencyProperty.RegisterAttached(
              "AnimatedSource",
              typeof(ImageSource),
              typeof(ImageBehavior),
              new UIPropertyMetadata(
                null,
                AnimatedSourceChanged));

        private static void AnimatedSourceChanged(DependencyObject o, DependencyPropertyChangedEventArgs e)
        {
            Image imageControl = o as Image;
            if (imageControl == null)
                return;

            var oldValue = e.OldValue as ImageSource;
            var newValue = e.NewValue as ImageSource;
            if (oldValue != null)
            {
                imageControl.BeginAnimation(Image.SourceProperty, null);
            }
            if (newValue != null)
            {
                imageControl.DoWhenLoaded(InitAnimationOrImage);
            }
        }

        private static void InitAnimationOrImage(Image imageControl)
        {
            BitmapSource source = GetAnimatedSource(imageControl) as BitmapSource;
            if (source != null)
            {
                var decoder = GetDecoder(source) as GifBitmapDecoder;
                if (decoder != null && decoder.Frames.Count > 1)
                {
                    var animation = new ObjectAnimationUsingKeyFrames();
                    var totalDuration = TimeSpan.Zero;
                    BitmapSource prevFrame = null;
                    FrameInfo prevInfo = null;
                    foreach (var rawFrame in decoder.Frames)
                    {
                        var info = GetFrameInfo(rawFrame);
                        var frame = MakeFrame(
                            source,
                            rawFrame, info,
                            prevFrame, prevInfo);

                        var keyFrame = new DiscreteObjectKeyFrame(frame, totalDuration);
                        animation.KeyFrames.Add(keyFrame);
                        
                        totalDuration += info.Delay;
                        prevFrame = frame;
                        prevInfo = info;
                    }
                    animation.Duration = totalDuration;
                    animation.RepeatBehavior = RepeatBehavior.Forever;
                    if (animation.KeyFrames.Count > 0)
                        imageControl.Source = (ImageSource)animation.KeyFrames[0].Value;
                    else
                        imageControl.Source = decoder.Frames[0];
                    imageControl.BeginAnimation(Image.SourceProperty, animation);
                    return;
                }
            }
            imageControl.Source = source;
            return;
        }

        private static BitmapDecoder GetDecoder(BitmapSource image)
        {
            BitmapDecoder decoder = null;
            var frame = image as BitmapFrame;
            if (frame != null)
                decoder = frame.Decoder;

            if (decoder == null)
            {
                var bmp = image as BitmapImage;
                if (bmp != null)
                {
                    if (bmp.StreamSource != null)
                    {
                        decoder = BitmapDecoder.Create(bmp.StreamSource, bmp.CreateOptions, bmp.CacheOption);
                    }
                    else if (bmp.UriSource != null)
                    {
                        Uri uri = bmp.UriSource;
                        if (bmp.BaseUri != null && !uri.IsAbsoluteUri)
                            uri = new Uri(bmp.BaseUri, uri);
                        decoder = BitmapDecoder.Create(uri, bmp.CreateOptions, bmp.CacheOption);
                    }
                }
            }

            return decoder;
        }

        private static BitmapSource MakeFrame(
            BitmapSource fullImage,
            BitmapSource rawFrame, FrameInfo frameInfo,
            BitmapSource previousFrame, FrameInfo previousFrameInfo)
        {
            DrawingVisual visual = new DrawingVisual();
            using (var context = visual.RenderOpen())
            {
                if (previousFrameInfo != null && previousFrame != null &&
                    previousFrameInfo.DisposalMethod == FrameDisposalMethod.Combine)
                {
                    var fullRect = new Rect(0, 0, fullImage.PixelWidth, fullImage.PixelHeight);
                    context.DrawImage(previousFrame, fullRect);
                }

                context.DrawImage(rawFrame, frameInfo.Rect);
            }
            var bitmap = new RenderTargetBitmap(
                fullImage.PixelWidth, fullImage.PixelHeight,
                fullImage.DpiX, fullImage.DpiY,
                PixelFormats.Pbgra32);
            bitmap.Render(visual);
            return bitmap;
        }

        private class FrameInfo
        {
            public TimeSpan Delay { get; set; }
            public FrameDisposalMethod DisposalMethod { get; set; }
            public double Width { get; set; }
            public double Height { get; set; }
            public double Left { get; set; }
            public double Top { get; set; }

            public Rect Rect
            {
                get { return new Rect(Left, Top, Width, Height); }
            }
        }

        private enum FrameDisposalMethod
        {
            Replace = 0,
            Combine = 1,
            RestoreBackground = 2,
            RestorePrevious = 3
        }

        private static FrameInfo GetFrameInfo(BitmapFrame frame)
        {
            var frameInfo = new FrameInfo
            {
                Delay = TimeSpan.FromMilliseconds(100),
                DisposalMethod = FrameDisposalMethod.Replace,
                Width = frame.PixelWidth,
                Height = frame.PixelHeight,
                Left = 0,
                Top = 0
            };

            BitmapMetadata metadata;
            try
            {
                metadata = frame.Metadata as BitmapMetadata;
                if (metadata != null)
                {
                    const string delayQuery = "/grctlext/Delay";
                    const string disposalQuery = "/grctlext/Disposal";
                    const string widthQuery = "/imgdesc/Width";
                    const string heightQuery = "/imgdesc/Height";
                    const string leftQuery = "/imgdesc/Left";
                    const string topQuery = "/imgdesc/Top";

                    var delay = metadata.GetQueryOrNull<ushort>(delayQuery);
                    if (delay.HasValue)
                        frameInfo.Delay = TimeSpan.FromMilliseconds(10 * delay.Value);

                    var disposal = metadata.GetQueryOrNull<byte>(disposalQuery);
                    if (disposal.HasValue)
                        frameInfo.DisposalMethod = (FrameDisposalMethod) disposal.Value;

                    var width = metadata.GetQueryOrNull<ushort>(widthQuery);
                    if (width.HasValue)
                        frameInfo.Width = width.Value;

                    var height = metadata.GetQueryOrNull<ushort>(heightQuery);
                    if (height.HasValue)
                        frameInfo.Height = height.Value;

                    var left = metadata.GetQueryOrNull<ushort>(leftQuery);
                    if (left.HasValue)
                        frameInfo.Left = left.Value;

                    var top = metadata.GetQueryOrNull<ushort>(topQuery);
                    if (top.HasValue)
                        frameInfo.Top = top.Value;
                }
            }
            catch (NotSupportedException)
            {
            }

            return frameInfo;
        }

        private static T? GetQueryOrNull<T>(this BitmapMetadata metadata, string query)
            where T : struct
        {
            if (metadata.ContainsQuery(query))
            {
                object value = metadata.GetQuery(query);
                if (value != null)
                    return (T) value;
            }
            return null;
        }

        #endregion
    }
```
  Et voici la méthode d'extension `DoWhenLoaded` utilisée dans le code ci-dessus :  
```csharp
public static void DoWhenLoaded<T>(this T element, Action<T> action)
    where T : FrameworkElement
{
    if (element.IsLoaded)
    {
        action(element);
    }
    else
    {
        RoutedEventHandler handler = null;
        handler = (sender, e) =>
        {
            element.Loaded -= handler;
            action(element);
        };
        element.Loaded += handler;
    }
}
```
   Cette classe sera inclue dans la prochaine version de la librairie [Dvp.NET](http://dvp-net.developpez.com/), dont j'avais [déjà parlé il y quelque temps](http://www.thomaslevesque.fr/2010/03/18/dvp-net-premiere-version-de-la-librairie-dvp-net/).  **Mise à jour** : le code qui récupère la durée d'une frame ne fonctionne que sous Windows Seven, et sous Windows Vista si la [Platform Update](http://support.microsoft.com/kb/971644/fr-fr) est installée (non testé). La durée par défaut (100ms) sera utilisée sur les autres versions de Windows. Je mettrai à jour l'article si je trouve une solution qui fonctionne sur tous les systèmes (je sais que je pourrais utiliser `System.Drawing.Bitmap`, mais je préfèrerais éviter...)  **Mise à jour 2** : comme Klaus l'a signalé dans un commentaire sur la version anglaise de mon blog, la classe `ImageBehavior` ne gérait pas certains attributs importants des frames : la méthode de destruction (est-ce qu'une frame doit simplement remplacer la frame précédente, ou être combinée avec elle), et la position des frames (Left/Top/Width/Height). J'ai mis à jour le code pour gérer ces attributs correctement. Merci Klaus !  **Mise à jour 3** : encore un petit bug corrigé, la récupération du décodeur à partir d'une URI relative ne fonctionnait pas. Merci à l'anonyme qui l'a signalé!  **Mise à jour 4** : plutôt que de continuer à poster les améliorations sur ce billet, j'ai finalement créé un [projet sur <strike>CodePlex</strike> GitHub](https://github.com/XamlAnimatedGif/WpfAnimatedGif/) où cette classe sera maintenue. Vous pouvez aussi l'installer avec NuGet, l'id du package est [WpfAnimatedGif](https://nuget.org/packages/WpfAnimatedGif). Merci à Diego Mijelshon pour la suggestion!

