---
layout: post
title: 'Transparent windows in WPF'
date: '2008-09-08'
categories:
  - WPF
published: true
---
## Introduction
WPF can obviously render transparent elements within its own window, but it also supports rendering the entire window with per-pixel transparency.  This feature comes with a few issues, which I'll discuss in this post.

## Layered Windows
Windows supports transparency at the HWND level through a feature called [layered windows](http://msdn.microsoft.com/en-us/library/ms632599(VS.85).aspx#layered).  Layered windows are represented by a bitmap, and the OS renders the bitmap whenever it needs to.  There are two layered windows modes currently supported by the OS: System-Redirected Content and Application-Provided Content.  In either case, the HWND is given the WS_EX_LAYERED extended style.  The mode is determined by which API you use to update the window.  Windows only supports the layered feature for top-level windows.  This is not a technique that can be used for child window transparency.

### System-Redirected Content
This mode is determined by using the [SetLayeredWindowAttributes](http://msdn.microsoft.com/en-us/library/ms633540(VS.85).aspx) API.  The bitmap for the window is allocated by the OS, and every attempt to render into a DC for the window is redirected to render into the bitmap.  The window receives WM_PAINT messages like normal, though the frequency of these messages is reduced since the OS does not need to have the application repaint the window as it moves around.  This mode provides an excellent level of compatibility with existing code.  However, because GDI does not preserve the alpha channel in all of its APIs, this mode does not support per-pixel transparency.  Instead, the SetLayeredWindowAttributes API lets you specify a color key and an opacity for the entire window. 
Note: on XP there was a bug where the redirection was not respected by DirectX.  This caused DX to render directly to the screen, while the OS assumed the window content was in a bitmap.  This caused lots of visual artifacts on the screen.  This issue was fixed in Vista.

### Application-Provided Content
This mode is determined by using the [UpdateLayeredWindow](http://msdn.microsoft.com/en-us/library/ms633556(VS.85).aspx) API.  The bitmap for the window is allocated by the application, and is passed to the OS which makes a copy.  The window does not receive WM_PAINT messages, since there is no need to ask the application to ever repaint itself.  If the application has fresh content, it is responsible for calling UpdateLayeredWindow again.  The application is completely responsible for generating its content.  This API allows the application to provide a bitmap that has a per-pixel alpha channel.  But because the OS does not redirect any GDI painting calls, this mode does not support compatibility with existing code.  

## WPF Chooses Application-Provided Content
WPF chose to only support the Application-Provided Content mode of layered windows.  This was chosen primarily because it enabled the broadest range of features, and we didn't want to confuse our API with a mix of modes and features.  WPF deeply supports per-pixel transparency in its rendering, so it was a natural fit.  Per-pixel transparency naturally allows WPF's anti-aliased rendering to work on the layered window too, which makes the edges look much nicer.
This choice has a few important ramifications:

### Non-Client Area
The "non-client" area of a window is a general reference to the parts of the window that the windowing system normally renders for the application.  This includes the title bar, the resize edges, the menu bar, the scroll bars, etc.  These parts are drawn by the windowing system to the DC of the window.  Because layered windows that use the application-provided content mode do not redirect GDI calls, the windowing system is not able to render into the bitmap that the application is using to represent its content.  Further, on Vista, the windowing system uses a complex shader to distort the background behind the non-client area.  This distortion cannot simply be encoded into a bitmap anyway.  And because the bitmap WPF is going to provide will represent the entire window (including the non-client area), WPF would have to carefully handle the non-client messages.  WPF chose to avoid all of this complexity by expanding the client area to fill the entire window by handling the WM_NCCALCSIZE message.  No matter what your window styles may suggest, transparent WPF windows do not have any visible non-client area.  This is fine for many scenarios where the intent is to create a custom window shape, but it can be annoying for people who just want to "fade in" a normal window.

### Child Windows
Again, because layered windows that use the application-provided content mode do not redirect GDI calls, child windows do not automatically get rendered into the bitmap that the application is going to use to represent its content.  You can try to get the child window to paint into a system-memory bitmap, and then blit the bitmap into WPF.  The problem with this is knowing when the child window is "dirty".  Most implementations use a timer to poll the content from the child window.  This is not efficient, but may be suitable for your needs.  A popular implementation of this technique is:
<http://www.codeplex.com/WPFWin32Renderer>

## Construction Only
Windows will let you switch a top-level window in and out of the various layered modes.  It is a bit tedious, as you have to clear the WS_EX_LAYERED extended style, then set it again, and call the appropriate API to lock in the new mode.  For simplicity, WPF chose not to support this ability.  You have to decide at construction time whether the window will be layered or not, and the window cannot be changed afterwards.  If needed, you can accomplish something similar by creating a new window, and transferring the element tree over to the new window.  WPF enforces this constraint rigidly, rejecting any attempt to alter the underlying WS_EX_LAYERED extended window style.

## HwndSource
You can specify that you want to use a layered window when you construct the HwndSource by setting the UsesPerPixelOpacity property of the HwndSourceParameters that you pass to the constructor.  When this setting is specified, the HwndSource class will set the WS_EX_LAYERED extended window style, prevent the window from being themed, and configures the WPF render target to have a transparent back buffer.

{% highlight C# %}
HwndSourceParameters p = new HwndSourceParameters("TestWindow", 100, 100); 
p.UsesPerPixelOpacity = true; 
p.WindowStyle |= 0x10000000; // WS_VISIBLE 
Ellipse ellipse = new Ellipse(); 
ellipse.Width = 100; 
ellipse.Height = 100; 
ellipse.Fill = Brushes.Red; 
ellipse.Opacity = 0.5; 
HwndSource hwndSource = new HwndSource(p); 
hwndSource.RootVisual = ellipse;
{% endhighlight %}

## Window
Of course, most people don't use HwndSource directly, but use the Window class instead.  The Window class offers its own property called AllowsTransparency.  The Window class delays creating the underlying HwndSource until it is shown, so this property is simply passed to the HwndSource constructor at that time.  Because the underlying HwndSource expands the client area to fill the entire window, the Window class also requires that the WindowStyle property be set to None to avoid confusion.
The Window class is a control with a template that renders a background, so if you often want to set the Background property to a brush with transparency.  Of course, you can also specify an opacity for the entire window by setting the Opacity property.
Here is some example XAML markup for create a layered window:

{% highlight XML %}
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation" 
  Width="100" 
  Height="100" 
  AllowsTransparency="True" 
  WindowStyle='None' 
  Background="Transparent"> 
    <Ellipse Width="100" Height="100" Fill="Red" Opacity="0.5"/> 
</Window>
{% endhighlight %}

## Hit Testing
There are two important phases to determining where to send mouse events:

### Which HWND is the mouse over?
The operating system must respond very quickly to mouse movement.  Windows uses a dedicated Raw Input Thread (RIT), running in the kernel, to handle the signals from the mouse hardware.  The RIT quickly scans over the HWND hierarchy to see which window the mouse is over.  Because the system must be responsive, no application code is invoked.  For normal windows, the RIT checks the mouse position against the window's rectangle (or region, if one is set).  But for layered windows, the RIT looks in the bitmap that specifies the content for the window and checks the effective transparency at that location (which can be affected by the constant opacity setting, the color key setting, or the per-pixel alpha channel).  If the pixel is 100% transparent, the RIT skips the window and keeps looking.  Once a window has been located, the mouse move flag is set on the thread that owns the window.  This will cause the thread to receive a WM_MOUSEMOVE message the next time it calls GetMessage() and there are no other higher-priority messages.

### Which UIElement is the mouse over?
WPF only uses an HWND as its outer container.  Within the HWND is a rich hierarchy of elements, and WPF determines which element to route the mouse events to via its own hit-testing.  In particular, WPF uses the UIElement.InputHitTest method.  WPF performs the hit-test against the geometry used for the rendering primitives.  If the geometry is filled, with any kind of brush, it is considered solid.  Even if the brush is "Transparent", WPF still considers the geometry to be solid, like a piece of glass.  On the other hand, if the geometry is not filled with a brush, it is considered hollow.  The important point here is that WPF does not check the transparency at the mouse position.  WPF hit-testing actually involves many other things - IsHitTestVisible, HitTestCore, Visibility, IsEnabled, etc - but those are best left to another article.
Also note that WPF routes the mouse events through the visual tree.  So mouse events will pass through all of the parent elements on the way to, and from, the destination element.  The event route is strictly based on the hierarchy of elements - it does not reflect what ordering of visuals you might see on the screen.
When using a layered window, you need to understand both hit-testing passes.  For WPF to receive the mouse messages at all, you must have at least a partially opaque pixel under the mouse.  If that condition is met, then the normal WPF rules apply.
Sometimes people want the top-level window to look transparent, but still receive mouse messages.  One technique is to use a nearly-transparent brush for your background.  This works pretty well on normal desktops, but will be noticeable on a lower color-depth; such as over terminal services.  Another technique is to grab mouse capture, but this is not a passive technique, and it will impact the user interactions with other windows.  Finally, you can install a mouse hook, but WPF does not expose any methods for this.  You would have to PInvoke to Win32 yourself.

## Performance
The performance of WPF's transparent windows is a topic of much concern.  WPF is designed to render via DirectX.  We do offer a software rendering fallback, but that is intended to provide a full-featured fallback, not for high performance.  The layered window APIs, on the other hand, take a GDI HDC parameter.  This causes different issues for XP and Vista.

## XP
DirectX does provide the [IDirect3DSurface9::GetDC](http://msdn.microsoft.com/en-us/library/bb205894(VS.85).aspx) method, which can return a DC that references the DirectX surface.  Unfortunately there was a restriction in DX9c that would fail this method if it were called on a surface that contained an alpha channel.  Of course, the entire point of our layered window API is to enable per-pixel transparency.  This restriction was lifted for Vista, but our initial release forced WPF to use its software rendering fallback with rendering to a layered window on XP.  We were able to lift this restriction for XP too, which we released as a hot fix ([KB 937106](http://support.microsoft.com/kb/937106/en-us)).  This hot fix was also included in XP SP3.  Now, on XP, we can render via DirectX and pass the results of IDirect3DSurface9::GetDC directly to UpdateLayeredWindow.  On good video drivers, the resulting copy will remain entirely on the video card, leading to excellent performance.  Some video drivers, however, may choose to perform this copy through system memory.  The performance on such systems will not be nearly as good, but should still be reasonable for many scenarios.

> Experience suggests that a full-screen constantly updating layered window on good XP machine can consume as little as 3% CPU in overhead.

## Vista
As already noted, Vista supports IDirect3DSurface9::GetDC on surfaces with an alpha channel.  The problem on Vista is that GDI is now completey emulated in software via the Canonical Display Device (CDD).  The CDD operates on system-memory buffers, so even though we could get a DC to our DX surface, blitting from it would cause the contents to be copied into system memory first.  The technique used for this copy was very inefficient.  We were able to improve this scenario by using the GetRenderTargetData API to fetch the entire contents into system memory, and then create a DC around the system memory before calling UpdateLayeredWindow.  The nature of this technique favors cards with higher GPU->CPU bandwidth, such as PCIe.  We released this as a hot fix (KB 938660).  This hot fix is also included in Vista SP1, so go get it!  While this technique is still slower on Vista than on XP, it should still be reasonable for many scenarios.

> Experience suggests that a full-screen constantly updating layered window on good Vista machine can consume as much as 30% CPU in overhead.  For performance improvements, reduce the amount of updates to the window, in either frequency or area.  WPF is optimized around dirty rect updates, so if you change just a portion of the window, WPF will be able to update just that portion to the screen.

## Bugs
We discovered a number of issues with WPF's use of layered windows when suspending/resuming, high CPU usage while fast user switching, layered windows not updating across a terminal services connection, minimized layered windows not updating when the desktop was locked, and crashes when changing the system color depth to 8bpp or less.  All known issues in these areas have been fixed in our 3.5 SP1 release.
We also discovered an issue where our call to UpdateLayeredWindow was failing with E_INVALIDARG.  The error resulted from the window no longer being in the Application-Provided Content mode.  But since WPF only allows the mode to be specified at construction time, and we only accept the Application-Provided Content mode, and we carefully reject attempts to change the relevant style bits at runtime, we were at a loss to explain how the window was changing modes.  It turns out that when you call PrintWindow, the OS will switch the window into System-Redirected Content mode so that it can grab a snap shot.  Since WPF renders on a separate thread, any use of PrintWindow on a WPF layered window was basically a race condition.  And it turns out that the IntelliPoint software was using PrintWindow to grab snapshots of all windows for its display.  IntelliPoint is widely installed, so any WPF apps that used layered windows were vulnerable to crashing.  While this is a real bug in the OS, we worked around it by simply trying to render again if the call to UpdateLayeredWindow fails in this manner.  This work-around is included in our 3.5 SP1 release.
On XP we also discovered that sometimes transparent windows would show up behind other windows, instead of in front.  This was a huge problem since in WPF menus, tooltips, and combo-box drop-downs are all implemented using transparent windows.  We eventually tracked this down to a layered window bug in XP, and a hotfix was made available.  This fix should be included in XP SP3.  However, we continue to here some (though much fewer) reports of the problem persisting on patched XP systems.  At this point we are unable to reproduce this behavior, so we do not have a fix.
