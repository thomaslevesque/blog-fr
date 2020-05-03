---
layout: post
title: '[WPF] Coller une image du presse-papier (bug dans Clipboard.GetImage)'
date: 2009-02-05T00:00:00.0000000
url: /2009/02/05/wpf-coller-une-image-du-presse-papier/
tags:
  - clipboard
  - image
  - presse-papier
  - WPF
categories:
  - Code sample
  - WPF
---


Hum... 2 mois depuis mon précédent (et premier) post... il faudra que j'essaie d'être un peu plus régulier à l'avenir ;-)

Si vous avez déjà essayé d'utiliser la méthode *Clipboard.GetImage* avec WPF, vous avez dû avoir une mauvaise surprise... En effet, cette méthode renvoie un *InteropBitmap* qui, dans certains cas (voire tout le temps), refuse de s'afficher dans un contrôle *Image* : aucune exception n'est levée, la taille de l'image est correcte, mais... l'affichage reste désespérément vide, ou alors l'image est méconnaissable.

Pourtant, si on enregistre l'image dans un stream et qu'on la relit à partir du stream, on obtient une image parfaitement utilisable. Mais bon, je trouve  cette méthode assez lourde (décodage - réencodage - redécodage de l'image). J'ai donc cherché une solution pour récupérer « manuellement » l'image dans le presse-papier.

Si on regarde les formats d'image disponibles dans le presse-papier (*Clipboard.GetDataObject().GetFormats()*), on voit que ça varie selon l'origine de l'image (capture d'écran, copie dans Paint…). Le seul format qui semble être toujours présent est *DeviceIndependentBitmap* (DIB). J'ai donc cherché, à récupérer le *MemoryStream* pour ce format, et à le décoder en *BitmapSource* :

```csharp
        private ImageSource ImageFromClipboardDib()
        {
            MemoryStream ms = Clipboard.GetData("DeviceIndependentBitmap") as MemoryStream;
            BitmapImage bmp = new BitmapImage();
            bmp.BeginInit();
            bmp.StreamSource = ms;
            bmp.EndInit();
            return bmp;
        }
```

Malheureusement, ce code renvoie une magnifique *NotSupportedException* : « Impossible de trouver un composant d'image adapté pour terminer l'opération.». En clair, il ne sait pas comment décoder ça… c'est pourtant un format assez simple a priori, et très répandu. J'ai donc un peu creusé la question et étudié la structure d'un DIB. En simplifiant un peu, un fichier bitmap « classique » (.bmp) est composé des sections suivantes :

- En-tête de fichier (structure *BITMAPFILEHEADER*)
- En-tête de bitmap (structure *BITMAPINFO*)
- Palette (tableau de RGBQUAD)
- Données de l'image


En observant le contenu du DIB dans le presse-papier, on voit qu'il présente la même structure, mais sans le *BITMAPFILEHEADER*… l'astuce est donc d'ajouter cet en-tête au début du buffer, et de passer le tout à *BitmapImage* ou *BitmapFrame*. Pas bien difficile a priori… là où ça se corse, c'est qu'il faut renseigner dans cet en-tête certaines valeurs qu'on ne peut obtenir qu'en lisant le contenu de l'image. Il faut notamment indiquer à quelle position débutent les données de l'image proprement dite, et donc connaitre la taille des en-têtes et de la palette. Le code suivant effectue ce traitement, et renvoie une *ImageSource* à partir du presse-papier :

```csharp
        private ImageSource ImageFromClipboardDib()
        {
            MemoryStream ms = Clipboard.GetData("DeviceIndependentBitmap") as MemoryStream;
            if (ms != null)
            {
                byte[] dibBuffer = new byte[ms.Length];
                ms.Read(dibBuffer, 0, dibBuffer.Length);

                BITMAPINFOHEADER infoHeader =
                    BinaryStructConverter.FromByteArray<BITMAPINFOHEADER>(dibBuffer);

                int fileHeaderSize = Marshal.SizeOf(typeof(BITMAPFILEHEADER));
                int infoHeaderSize = infoHeader.biSize;
                int fileSize = fileHeaderSize + infoHeader.biSize + infoHeader.biSizeImage;

                BITMAPFILEHEADER fileHeader = new BITMAPFILEHEADER();
                fileHeader.bfType = BITMAPFILEHEADER.BM;
                fileHeader.bfSize = fileSize;
                fileHeader.bfReserved1 = 0;
                fileHeader.bfReserved2 = 0;
                fileHeader.bfOffBits = fileHeaderSize + infoHeaderSize + infoHeader.biClrUsed * 4;

                byte[] fileHeaderBytes =
                    BinaryStructConverter.ToByteArray<BITMAPFILEHEADER>(fileHeader);

                MemoryStream msBitmap = new MemoryStream();
                msBitmap.Write(fileHeaderBytes, 0, fileHeaderSize);
                msBitmap.Write(dibBuffer, 0, dibBuffer.Length);
                msBitmap.Seek(0, SeekOrigin.Begin);

                return BitmapFrame.Create(msBitmap);
            }
            return null;
        }
```

Définition des structures *BITMAPFILEHEADER* et *BITMAPINFOHEADER* :

```csharp
        [StructLayout(LayoutKind.Sequential, Pack = 2)]
        private struct BITMAPFILEHEADER
        {
            public static readonly short BM = 0x4d42; // BM

            public short bfType;
            public int bfSize;
            public short bfReserved1;
            public short bfReserved2;
            public int bfOffBits;
        }

        [StructLayout(LayoutKind.Sequential)]
        private struct BITMAPINFOHEADER
        {
            public int biSize;
            public int biWidth;
            public int biHeight;
            public short biPlanes;
            public short biBitCount;
            public int biCompression;
            public int biSizeImage;
            public int biXPelsPerMeter;
            public int biYPelsPerMeter;
            public int biClrUsed;
            public int biClrImportant;
        }
```

Classe utilitaire pour convertir des structures en binaire :

```csharp
    public static class BinaryStructConverter
    {
        public static T FromByteArray<T>(byte[] bytes) where T : struct
        {
            IntPtr ptr = IntPtr.Zero;
            try
            {
                int size = Marshal.SizeOf(typeof(T));
                ptr = Marshal.AllocHGlobal(size);
                Marshal.Copy(bytes, 0, ptr, size);
                object obj = Marshal.PtrToStructure(ptr, typeof(T));
                return (T)obj;
            }
            finally
            {
                if (ptr != IntPtr.Zero)
                    Marshal.FreeHGlobal(ptr);
            }
        }

        public static byte[] ToByteArray<T>(T obj) where T : struct
        {
            IntPtr ptr = IntPtr.Zero;
            try
            {
                int size = Marshal.SizeOf(typeof(T));
                ptr = Marshal.AllocHGlobal(size);
                Marshal.StructureToPtr(obj, ptr, true);
                byte[] bytes = new byte[size];
                Marshal.Copy(ptr, bytes, 0, size);
                return bytes;
            }
            finally
            {
                if (ptr != IntPtr.Zero)
                    Marshal.FreeHGlobal(ptr);
            }
        }
    }
```

L'image obtenue par ce code peut être utilisée sans problème dans un contrôle *Image*.

Comme quoi, WPF a beau être une technologie « dernier cri », il faut encore parfois mettre les mains dans le cambouis ! Espérons que Microsoft corrigera ce bug dans la prochaine version…

