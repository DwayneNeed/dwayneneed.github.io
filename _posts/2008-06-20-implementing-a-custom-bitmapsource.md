---
layout: post
title: 'Implementing a custom BitmapSource'
date: '2008-06-20'
categories:
  - WPF
published: true
---
![screenshot of demo application](/static/img/2008-06-20-implementing-a-custom-bitmapsource_demo-screenshot.png)

## Windows Imaging Component
WPF uses the Windows Imaging Component (WIC) library to process bitmap data. WIC presents a data flow for bitmap processing that is both simple and powerful. The basic interface for bitmap processing in WIC is IWICBitmapSource. This interface provides a common way of accessing and linking together bitmaps, decoders, format converters, and other bitmap processing components. Components that implement this interface can be connected together in a graph to pull imaging data through.  For example:

![screenshot of demo application](/static/img/2008-06-20-implementing-a-custom-bitmapsource_bitmap-pipeline.png)

Once the bitmap processing graph is configured, the data flow is initiated by calling CopyPixels on the component at the front of the graph.  CopyPixels instructs the component to produce pixels according to its algorithm - this may involve decoding a portion of a JPEG stored on disk, copying a block of memory, or even analytically computing a complex gradient. The algorithm is completely dependent on the object implementing the interface.  In the most common scenario, bitmap processing components are chained together, as shown in the example above.  In such scenarios, the components typically call CopyPixels on the component it is connected to to get the actual bitmap data, and then performs some computation on the results.

WIC provides a rich set of components that provide important functionality.  These include encoders and decoders for popular image formats (BMP, GIF, ICO, JPEG, PNG, TIFF, HDPhoto), pixel format converters, color space transforms, clippers, scalers, etc. 

## WIC in WPF
WPF uses WIC internally to implement various features, and it also exposes much of the WIC API through its imaging classes.  The base class for bitmap processing components in WPF is BitmapSource.  The standard WIC components are exposed as derived classes.

Many of the WPF imaging classes implement the ISupportInitialize interface.  On instances of these classes, properties can only be set between calls the BeginInit and EndInit.  Setting properties after EndInit is called have no effect.  The non-default constructors typically call BeginInit and EndInit appropriately.  The default constructors do not call these methods, and you must call them yourself.  This allows you to set the properties yourself.  Neglecting to call EndInit will prevent any bitmap processing.

Some of the WPF imaging classes accept URIs that identify the location of bitmap data.  These URIs can specify data embedded in the application, on the local disk, and on the network.  WPF will download the bitmap data for a network location on a background thread.  This helps prevent the UI thread from blocking while downloading.  The imaging classes that accept URIs also implement the IUriContext interface.  This interface allows a BaseUri to be specified in order to resolve any relative URIs.  In many cases WPF will set the BaseUri automatically.

WPF implements an aggressive caching policy to avoid duplicate work.  In addition to the standard WinINet cache, both bitmaps and decoders are cached, allowing multiple references to the same Uri to share the same bitmap instance and avoid redundant decoding.  There are flags that can be used to bypass the caches.

The WPF rasterizer only natively renders bitmaps in Bgr32 and PBgra32 formats.  Even though the WPF imaging classes can process bitmap data with different formats, they are converted to either Bgra32 or PBgra32 before being rendered.  There are certainly scenarios that could benefit from other pixel formats, either for less memory usage (such as black/white formats) or for higher color fidelity (such as 16-bit grayscale); however WPF does not yet provide rendering support for them.

Finally, most of the WPF imaging classes are sealed.  This design choice was made to ensure a higher quality product, but it interferes with many legitimate scenarios.  We will discuss this more later.

Let's take a moment to review the imaging classes in WPF:

![screenshot of demo application](/static/img/2008-06-20-implementing-a-custom-bitmapsource_class-hierarchy.png)

#### [BitmapSource](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.bitmapsource.aspx)
This is the abstract base class of the WPF bitmap classes.  BitmapSource class does not directly implement the IWICBitmapSource interface.  Instead, it is a class designed to wrap an IWICBitmapSource implementation.  The BitmapSource class has virtual members that provide the same basic functionality as IWICBitmapSource: CopyPixels, Format, PixelWidth, PixelHeight, DpiX, DpiY, and Pallete.

#### [BitmapFrame](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.bitmapframe.aspx)
Represents image data returned by a decoder and accepted by encoders.  You can call one of the static Create methods to create an instance from a URI, a Stream, or an existing BitmapSource.  Creating a BitmapFrame will automatically assemble the imaging components needed to decode the data.

#### [FormatConvertedBitmap](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.formatconvertedbitmap.aspx)
This class wraps the standard WIC pixel format converter (IWICImagingFactory::CreateFormatConverter).  This component provides the means of converting from one pixel format to another, handling dithering and halftoning to indexed formats, palette translation and alpha thresholding. The Source, DestinationFormat, DestinationPalette, and AlphaThreshold properties are used to initialize the underlying component via IWICFormatConverter::Initialize.  The dither type is hard-coded to be WICBitmapDitherTypeErrorDiffusion.  The palette translation type is hard-coded to be WICBitmapPaletteTypeMedianCut. The ISupportInitialize interface is used to snap the values of the properties when initialization is complete.  Further changes to the properties are ignored.

#### [ColorConvertedBitmap](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.colorconvertedbitmap.aspx)
This class wraps the standard WIC color transformer (IWICImagingFactory::CreateColorTransformer).  This component transforms the color space of the input bitmap source.  The Source, SourceColorContext, DestinationColorContext, and DestinationFormat properties are used to initialize the underlying component via IWICColorTransform::Initialize. The ISupportInitialize interface is used to snap the values of the properties when initialization is complete.  Further changes to the properties are ignored.

#### [TransformedBitmap](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.transformedbitmap.aspx)
This class wraps both the standard WIC scaler (IWICImagingFactory::CreateBitmapScaler) and flip-rotator (IWICImagingFactory::CreateBitmapFlipRotator). The scaler component produces a resized version of the input bitmap source using a resampling or filtering algorithm. The interpolation mode is hard-coded to be WICBitmapInterpolationModeFant.  The flip-rotator component produces a flipped (horizontal or vertical) and/or rotated (by 90 degree increments) version of the input bitmap source. Rotations are done before the flip. The Transform property is used to specify both the scale and the rotation to apply to the bitmap source.  Along with the Source property, this data is used to initialize to the underlying components via IWICBitmapScaler::Initialize and/or IWICBitmapFlipRotator::Initialize.  The ISupportInitialize interface is used to snap the values of the properties when initialization is complete.  Further changes to the properties are ignored.

#### [CroppedBitmap](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.croppedbitmap.aspx)
This class wraps a standard WIC clipper (IWICImagingFactory::CreateBitmapClipper). This component produces a clipped version of the input bitmap source for a specified rectangular region of interest.  The Source and SourceRect properties are used to initialize the underlying component via IWICBitmapClipper::Initialize. The ISupportInitialize interface is used to snap the values of the properties when initialization is complete.  Further changes to the properties are ignored.

#### [CachedBitmap](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.cachedbitmap.aspx)
This class represents bitmap data that is cached in system memory.  You can call one of the static BitmapSource.Create methods to create an instance from a memory buffer (IWICImagingFactory::CreateBitmapFromMemory).  The contents of the memory are copied into the CachedBitmap instance, and changes to the memory do not change the contents of the cached bitmap.  You can also call the CachedBitmap constructor and pass an bitmap source to cache (IWICImagingFactory::CreateBitmapFromSource).  The contents of the bitmap source are copied into the CachedBitmap instance, and changes to the bitmap source do not change the contents of the cached bitmap.

#### [BitmapImage](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.bitmapimage.aspx)
This class is the main WPF imaging class for downloading and decoding a bitmap.  The ISupportInitialize interface is used to snap the values of the properties when initialization is complete.  Further changes to the properties are ignored. This class is very complex, and offers the following functionality:

* **Loading from a stream**  
To load bitmap data from a stream, set the StreamSource property.
* **Loading from a Uri**  
To load bitmap data from a Uri, set the UriSource property.  BitmapImage implements IUriContext to facilitate the handling of relative Uris.  To control how the Uri is fetched from the WinINet cache, set the UriCachePolicy property.  For convenience in XAML, there is a custom type converter to convert from the HttpRequestCacheLevel enumeration.
* **Downloading**  
BitmapImage will download bitmap data from a network location on a background thread to avoid blocking the UI thread.  The IsDownloading property will indicate if the data is being downloaded.  The DownloadCompleted, DownloadFailed, and DownloadProgress events are raised as appropriate.  Note that the DownloadCompleted event is not raised if no downloading occurred.
* **Decoding**  
The bitmap data can be of any format, and WIC will search for an appropriate decoder. For efficiency, you can specify the DecodePixelWidth and DecodePixelHeight properties.  These properties will control the size of the decoded image in memory.  While downloading is done on a background thread, decoding is done on the UI thread.
* **Caching**  
BitmapImage will store the decoded bitmap in system memory.  You can control when this happens by setting the CacheOption property.  BitmapImage also maintains a cache of previous BitmapImage instances (via weak references) so that loading the same Uri multiple times will share the same instance.  To avoid this cache, you can include the BitmapCreateOptions.IgnoreImageCache flag in the CreateOptions property.
* **Cropping & Rotating**  
The SourceRect property can be set to indicate the area of the bitmap to crop, and the Rotation property can be set to specify a rotation to apply.  Certain decoders can implement these operations when decoding.

#### [InteropBitmap](http://msdn.microsoft.com/en-us/library/system.windows.interop.interopbitmap.aspx) 
This class wraps standard WIC components for HICONs (IWICImagingFactory::CreateBitmapFromHICON), HBITMAPs (IWICImagingFactory::CreateBitmapFromHBITMAP), and shared memory sections(WICCreateBitmapFromSection).  These bitmap sources are primarily used in interop scenarios. To create a bitmap source for an HICON, call Imaging.CreateBitmapSourceFromHIcon.  To create a bitmap source for an HBITMAP, call Imaging.CreateBitmapSourceFromHBitmap.  To create a bitmap source around a shared memory section, call Imaging.CreateBitmapSourceFromMemorySection.  In all cases, the Imaging methods return a BitmapSource instance, even though the instance is really an InteropBitmap.  This is generally not a problem since InteropBitmap does not offer any additional API.  One exception is the Invalidate method.  This method only works for InteropBitmap instances created around shared memory, and indicates that WPF should reload the bitmap from the data in the shared memory section.  To call this method, you must first up-cast to InteropBitmap.

#### [WriteableBitmap](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.writeablebitmap.aspx)
This class represents a bitmap that can be written to by the application via the WritePixels method.  In previous versions of WPF, this method would end up allocating a new bitmap on every update.  We have improved this class significantly WPF 3.5 SP1, due out later this summer.  Improvements include efficient double buffering, direct access to the back buffer, dirty rects, sub-byte pixel formats, greater flexibility with array types, and more!  This class is supported in partial trust.  Having addressed the major performance issues, we expect that this class will enable some really cool scenarios.

#### [RenderTargetBitmap](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.rendertargetbitmap.aspx)
This class represents a bitmap produced by rasterizing WPF visuals.  You specify the bitmap dimensions, pixel format, etc when you construct an instance.  Currently RenderTargetBitmap only supports Pbgra32. Simply call Render and pass the visual you want to rasterize into the bitmap.  You can also call Clear to clear the bitmap if needed.

## Static Vs Dynamic Bitmaps
The WPF imaging classes are primarily designed for static image processing - such as loading bitmaps.  Once a bitmap has been processed, the results are cached in memory.  Any further changes to the imaging components are ignored.  This poses a problem for bitmap data that is asynchronous.  As noted above, WPF downloads bitmap data from network locations on a background thread.  It is possible that when WPF renders and calls through IWICBitmapSource.CopyPixels that the bitmap data will not be available.  The call to CopyPixels is just a no-op, and the empty results are cached.  Even when the download finishes, the bitmap is not updated since there is already a cache of it in memory.

The problem can be seen with this simple XAML:

{% highlight XML %}
<Page xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation" xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"> 
  <Image> 
    <Image.Source> 
      <TransformedBitmap> 
        <TransformedBitmap.Source> 
          <BitmapImage UriSource="http://www.microsoft.com/presspass/presskits/windowsvista/images/icons/IE.jpg"/> 
        </TransformedBitmap.Source> 
      </TransformedBitmap> 
    </Image.Source> 
  </Image> 
</Page>
{% endhighlight %}

The first time you load this XAML, there is a good chance that you won't see the image.  Change the UriSource to some other "fresh" URI to reproduce the problem, because once WPF has downloaded the image data, it will always work (until you restart the app) since there is no background thread download needed anymore.

The WPF way of doing this would be to listen for the DownloadCompleted event (only if IsDownloading is true) and waiting to hook up the TransformedBitmap until the download completes.  While this is certainly doable by an application, it is rather non-obvious and annoying, and it interferes with expressing the bitmap processing chain declaratively.

Instead of providing a fully dynamic bitmap processing pipeline, WPF bakes support for common scenarios into the BitmapImage class itself.  There are various properties on BitmapImage you can set, and then BitmapImage will construct the chain of appropriate WIC bitmap sources for you when the download completes.  Indeed, BitmapImage is your one-stop shop for assembling the most common standard WIC components.  Depending on the properties specified, BitmapImage will assemble the following chain:

![screenshot of demo application](/static/img/2008-06-20-implementing-a-custom-bitmapsource_demo-bitmap-pipeline.png)

WPF is also very careful about running arbitrary code on the render thread.  By caching the result of bitmap processing, the heavy work (such as decoding the bitmap) is performed on the UI thread, and not on the render thread.

Even though the WPF imaging classes are primarily designed for static image processing, there are a few imaging classes designed for dynamic bitmap scenarios: WriteableBitmap, RenderTargetBitmap, and an InteropBitmap created around a shared memory section.  These classes are specially handled by WPF for dynamic content.  In our forth-comming 3.5 SP1 release, the WriteableBitmap implementation has been substantially improved to provide a reasonably efficient, double-buffered, constant-memory solution for dynamically updating the contents of a bitmap.

## Custom BitmapSources
While most of the imaging classes are sealed, WPF does actually support external classes that derive from BitmapSource.  In this case, WPF will create an internal implementation of IWICBitmapSource that simply calls the virtual methods on BitmapSource.  WPF adds some additional virtuals, including overloads of CopyPixels, and some events for downloading and decoding.  All derived classes must override all of these virtuals, regardless of whether or not you need them.  In particular, the derived class must supply an implementation for:

* [virtual PixelFormat Format {get;}](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.bitmapsource.format.aspx)
* [virtual int PixelWidth {get;}](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.bitmapsource.pixelwidth.aspx)
* [virtual int PixelHeight {get;}](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.bitmapsource.pixelheight.aspx)
* [virtual double DpiX {get;}](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.bitmapsource.dpix.aspx)
* [virtual double DpiY {get;}](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.bitmapsource.dpiy.aspx)
* [virtual BitmapPalette Palette {get;}](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.bitmapsource.palette.aspx)
* [virtual void CopyPixels(Int32Rect sourceRect, Array pixels, int stride, int offset);](http://msdn.microsoft.com/en-us/library/ms616042.aspx)
* [virtual void CopyPixels(Array pixels, int stride, int offset);](http://msdn.microsoft.com/en-us/library/ms616043.aspx)
* [virtual void CopyPixels(Int32Rect sourceRect, IntPtr buffer, int bufferSize, int stride);](http://msdn.microsoft.com/en-us/library/ms616044.aspx)  
Important note: because your custom bitmap class has to override the CopyPixels method that handles a raw memory pointer it can only run in full trust.
* [virtual bool IsDownloading {get;}](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.bitmapsource.isdownloading.aspx)
* [virtual event EventHandler DownloadCompleted {add; remove;}](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.bitmapsource.downloadcompleted.aspx)
* [virtual event EventHandler<DownloadProgressEventArgs> DownloadProgress {add; remove;}](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.bitmapsource.downloadprogress.aspx)
* [virtual event EventHandler<ExceptionEventArgs> DownloadFailed {add; remove}](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.bitmapsource.downloadfailed.aspx)
* [virtual event EventHandler<ExceptionEventArgs> DecodeFailed {add; remove;}](http://msdn.microsoft.com/en-us/library/system.windows.media.imaging.bitmapsource.decodefailed.aspx)

## Introducing Microsoft.DwayneNeed.Media.Imaging
Most code from my blog gets posted to my codeplex site: http://www.codeplex.com/MicrosoftDwayneNeed Feel free to download it and play with it, or even just link it into an app if you find it useful.  For this blog I have added some custom bitmap classes:

![screenshot of demo application](/static/img/2008-06-20-implementing-a-custom-bitmapsource_custombitmap-class-hierarchy.png)

#### CustomBitmap 
This class implements the required virtuals from BitmapSource and provides reasonable default values.  For example, the DPI is returned as 96.0, the resolution is 0 x 0, the format is Bgr32, etc.  The three flavors of CopyPixels are sealed and redirected to a single CopyPixelsCore method.  This is the main method you will need to override to implement your logic.  There are some virtual members that make no sense at this level, such as IsDownloading and the various downloading and decoding events.  CustomBitmap simply provides storage for the event handlers and provides a protected method for raising the events.  You will need to call these protected raise methods if you actually implement downloading or decoding functionality.  A BitmapSource inherits from Freezable (eventually), and the freezable contract can be tedious to implement correctly.  CustomBitmap implements CreateInstanceCore by using reflection to create a new instance of itself.  This should be sufficient for most scenarios, and you don't need to provide an implementation in your derived classes unless you want to provide a more performant implementation or if you need to call a non-default constructor.  Further, the four "copy" virtual functions CloneCore, CloneCurrentValueCore, GetAsFrozenCore, and GetCurrentValueAsFrozenCore are all sealed and redirected to a single CopyCore method.  You will need to override this method if you have state that is not stored in DependencyProperties and that needs to be copied to freshly cloned instances.  You will also need to override FreezeCore if your additional state needs to be made readonly when the instance is frozen.  If your state is stored in DependencyProperties, the base Freezable implementation will correctly copy and freeze it as needed.

#### ChainedBitmap 
This class represents a custom bitmap that will be chained such that it processes the output of another bitmap source. The default implementation of the BitmapSource virtual members is to delegate to the chained bitmap source.  This makes sense for most properties like DpiX, DpiY, PixelWidth, PixelHeight, etc, as in many scenarios these properties are the same for the entire chain of bitmap sources.  However, derived classes should pay special attention to the Format property.  Many bitmap processors only support a limited number of pixel formats, and they should return this for the Format property.  ChainedBitmap will take care of converting the pixel format as needed in the base implementation of CopyPixels.  ChainedBitmap also adds handlers to the download and decode events of the chained bitmap source and raises its own events in response.  This allows a download or decode event to propagate through the chain of bitmap sources.

#### ColorKeyBitmap 
This class processes a source and produces Bgra32 bits, where all source pixels are opaque except for one color that is transparent.  You can specify this color explicitly be setting the TransparentColor property.  If this property is not set, the color of the pixel at (0,0) will be used as the transparent color.  For simplicity, we only process Bgra32 formatted bitmaps, anything else is converted to Bgra32.  Converting a pixel format that does not have an alpha channel to Bgra32 simply creates an opaque alpha value.  ColorKeyBitmap then changes the alpha channel for pixels that match the transparent color.

#### SepiaBitmap 
This class processes a source and converts the image to a sepia color scheme.  The algorithm is taken from http://msdn.microsoft.com/en-us/magazine/cc163866.aspx, and operates in the linear scRGB color space.  For simplicity, we only process Bgr32 or Bgra32 formatted bitmaps, anything else is converted to one of those formats.  Bgr32 and Bgra32 share the same memory layout for the RGB channels.

#### GrayscaleBitmap 
This class processes a source and converts the image to a grayscale color scheme.  The algorithm is taken from http://en.wikipedia.org/wiki/Grayscale, and operates in the linear scRGB color space.  For simplicity, we only process Bgr32 or Bgra32 formatted bitmaps, anything else is converted to one of those formats.  Bgr32 and Bgra32 share the same memory layout for the RGB channels.

## Demo
I created a simple demo and included it on the codeplex site.  It simply allows you to select a URI (I provide a list of images from www.microsoft.com, or you can enter your own), and then chain together up to two of the custom bitmap sources.  The app handles the case where the BitmapImage may still be downloading, and defers creating the chain until the content is ready.  Make your own CustomBitmap classes and try them out!

#### Stay Tuned...
Next we'll turn our attention to addressing the static processing limitations of the built-in WPF imaging classes.
