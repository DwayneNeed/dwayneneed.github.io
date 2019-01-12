---
layout: post
title: 'Blurry Bitmaps'
date: '2007-10-05'
categories:
  - WPF
published: true
---
![Blurry images](https://raw.githubusercontent.com/dwayneneed/dwayneneed.github.io/master/static/img/_posts/blurrybitmaps_thumb.png)

## Background: resolution independence

WPF was designed from the start to be resolution independent.  So instead of designing your UI in terms of pixels, you are encouraged to use a physical measuring unit (like inches).  When you specify coordinates in WPF, you are actually using "measure units", which are defined to be 1/96th of an inch.  This was chosen because 96 DPI monitor settings are very common.

When WPF renders to the screen, it takes into account the system DPI, which you can set from the display control panel.  Of course, one problem with this is that no one ever sets this correctly.  When was the last time you took out a ruler, measured your screen, then divided your resolution by the result?  Another reason people don't set the DPI correctly is that Windows doesn't always look good at non-standard DPI settings.  This is something we will be improving in the future.  Of course, since you have to recalculate the DPI settings everytime you adjust your resolution, it would be best if the hardware could just report it.  For these reasons, WPF may not actually be able to draw a real inch on the screen, but it should come reasonably close.

Another aspect of resolution independence is being able to draw with more precision than pixels.  If you recall the GDI APIs, drawing calls specified their coordinates as integers, which match the pixel grid.  WPF specifies its coordinates using floating point numbers, so you can easily specify coordinates like (1.2,5.234).  This is sometimes referred to as sub-pixel positioning.  Of course, what does it mean to render smaller than a pixel, given that each pixel has a single final color?  The answer is that we blend all of the contributors to a given pixel.  You can think of this as a weighted average, so that if one primitive covers most of a pixel, it will contribute more than other primitives that cover very little of the pixel.  But blending loses detail, so this can be very noticeable.

### Bitmaps: what size should they be?

Bitmaps are often produced by designers using high-end editing tools.  Presume that the goal is to preserve the designer's intent, what size should we display the image at?  If the designer made a 96x96 bitmap, on a 96 DPI machine it will be 1 inch by 1 inch.  But if you show that bitmap on a super-high-end 300 DPI monitor (I've seen one, its awesome), then the bitmap would be less than a centimeter in each dimension. So, did the designer mean 96 pixels, or one inch?  Interestingly, modern image formats allow you to embed the designer's intended DPI into the file.  WPF will utilize this information, and scale the bitmap so that it ends up being the desired size.  However, here we have a problem - WPF always presumes the designer meant the image to be a certain physical size.  Often designers mean for the image to be a certain pixel size - after all, they are carefully assigning colors to each pixel in the image.  Its not the designer's intent to be a certain physical size, but the tools they use often stamp some DPI setting into the file anyways (commonly, some standard DPI setting like 72 or 96).  WPF sees this DPI information and scales the image to match.

### One problem: pixel scaling

Bitmaps are difficult because they encode "high frequency" information in the form of per-pixel color information.  If the bitmap is displayed at a slightly different size, the result can be very blurry.  I won't bore you with lots of examples, but consider this one.  Say you have a bitmap that is 3x3 pixels.  It looks like this:

![Clear red bar](https://raw.githubusercontent.com/dwayneneed/dwayneneed.github.io/master/static/img/_posts/redbar_thumb_1.png)

Now, if we scale this just a bit, such that the image will fit into 4x4 pixels, the result suffers from the fact that we can't represent each source pixel equally in the destination grid.  In fact, each original pixel "expands" by 1/3.  But of course, the final pixel can only have one color, so we have to combine overlapping samples.  We can try various blending techniques, but they all have problems.  The result will look something like this:

![Blurry pink bar](https://raw.githubusercontent.com/dwayneneed/dwayneneed.github.io/master/static/img/_posts/pinkbar_thumb.png)

The human eye can easily detect this artifact, and we tend to associate the result with being out of focus, or blurry.  It can be very irritating.

### Another problem: pixel alignment

As I mentioned in the background section, WPF can render with sub-pixel precision.  What happens if you render the original 3x3 bitmap at the correct size, but at pixel coordinates (0.33,0.33)?  The solid red line ends up straddling two pixel columns, and the best we can do is to blend the contributions, which will result in nearly the same result as the previously described scale.

### SnapsToDevicePixels

WPF anticipated that there would be cases where people wanted to align with the pixel grid instead of using sub-pixel precision.  You can set the SnapsToDevicePixels property on any UIElement.  This will cause us to try and render to the pixel grid, but there are quite a few cases that don't work - including images.  We will be looking to improve this in the future.

### Now what?

All is not lost.  Even though WPF wants really badly to be resolution independent, we can force it to be pixel aligned ourselves.  In fact, this isn't even very hard.  We just need to do two things:

1. Size ourselves to real pixel sizes.
1. Position ourselves on pixel boundaries.

For sizing, we can easily participate in the measure pass, and return a measure size that is equivalent to the desired pixel size.  The transform used to factor in the system DPI is provided for us in CompositionTarget.TransformFromDevice.  The pixel size is in "device" coordinates, and we transform from the device into measure units.  Easy enough.

For positioning, we could participate in the arrange pass, and maybe apply a render transform to ourselves to offset appropriately.  However, for this example I'm focusing on bitmaps, so I choose to participate in rendering and pass the appropriate offset to the DrawImage call.  Either way would work, but the real problem is that we need to know when anything might shift our position.  The WPF layout system is very conservative, meaning that it tries to do as little work as possible.  Once you have been measured and arranged, you won't get called again unless your individual layout state is invalidated.  If some parent decides to shift you around a little bit, you will most likely not be laid out again.  I work around this by subscribing to the LayoutUpdated event.  Even though this event is an instance event on UIElement, it actually gets raised anytime any layout activity happens.  Lucky for us, this is pretty much what we want.

## Bitmap class

The code included below introduces a new class called Bitmap.  Bitmap is a replacement for Image, but instead of displaying any image source, it only displays bitmap sources.  This lets me access the PixelWidth and PixelHeight properties for determining the appropriate size.  The important aspects of this class are:

* Derived from UIElement instead of FrameworkElement because I don’t want things like MinWidth, MaxWidth, or even Width.
* Bitmap.Source can be set to any BitmapSource.
* When measured, it will return the appropriate measure units to display the bitmap’s PixelWidth and PixelHeight
* When rendered, it will offset the image it draws to align with the pixel grid
* Whenever layout updates, it checks to see if it needs to re-render to align to the pixel grid again.

Its a pretty straight-forward class, check out the source code for the details.

### The sample application

Included in the project is a very simple sample application.  I have embedded a number of small bitmaps with high-frequency data in them.  Further, I encoded the bitmaps are various designer DPIs.  In the app, the top stack panel contains instances of Bitmap.  The bottom stack panel contains instances of Image.  You can clearly see how Image responds to the bitmap DPI, while Bitmap does not.  For fun, you can use the arrow keys which will adjust a render transform on the root.  Bitmap will snap to the nearest pixel, while Image will use sub-pixel positioning.

### Is there a down side?

Of course.  By using the pixel size of the bitmap, the bitmap will look very different on different machines.  The 300 DPI super monitor will make your bitmap look very small.  Further, since we align to the pixel boundary, animating will cause the bitmap to jump by full pixels.  Sub-pixel positioning is very good for animations.  Finally, you can get gaps between elements due to rounding to the pixel grid.  If you can live with these limitations, then maybe this Bitmap class will be useful to you.
