---
layout: post
title: 'Multithreaded UI: HostVisual'
date: '2007-04-26'
categories:
  - WPF
published: true
---

## Background: The WPF Threading Model

In general, objects in WPF can only be accessed from the thread that created them.  Sometimes this restriction is confused with the UI thread, but this is not true, and it is perfectly fine for objects to live on other threads.  But it is not generally possible to create an object on one thread, and access it from another.  In almost all cases this will result in an InvalidOperationException, stating that **“The calling thread cannot access this object because a different thread owns it.”**

## Freezables

Of course, there are exceptions to this restriction.  A familiar one is the Freezable class.  Freezable objects can be frozen, at which point they become read-only and we lift the single-thread restriction.  Examples of frozen Freezables include the standard brushes, available from the Brushes class.  These brushes can be used on any thread at any time.

This is incredibly useful, and used for all of the graphics primitive resources (pens, brushes, transforms, etc).  You can even derive your own types from Freezable and play by the same rules.

## Separate Windows

Unfortunately, that read-only restriction can be a real problem.  There are many scenarios where we want to run separate pieces of the UI on separate threads.  If these pieces of UI are independent from each other, you can host them in separate windows, and run the windows on separate threads.  This can be a reasonable solution for some scenarios, especially scenarios where the separate threads can be run in independent top-level windows.  The big restriction for this approach is that the graphics from one cannot be composited with the graphics of the other.  So while you could use a child window for the other thread’s UI, the system will just render one on top of the other.  You cannot have transparency, you cannot use one as a brush for 3D content, you can’t draw over it, etc.

## HostVisual

If your scenario doesn’t require interactivity (meaning input), then there is another option that WPF provides: HostVisual.  This option leverages the powerful composition engine in WPF that is already capable of aggregating the rendering primitives from multiple threads into one scene.  The element tree owned by the worker thread is rendered into its own composition target (called VisualTarget), and the results are composed into the HostVisual owned by the UI thread.

## Issue #1: Hosting a Visual in XAML

The first problem to solve is that the HostVisual class derives from Visual.  I can't use an existing panel, such as Border, to host this visual.  Border derives from Decorator, which is the standard base class for panels that have a single child.  Unfortunately, the child is strongly typed to be a UIElement.  I have to use a HostVisual, which does not derive from UIElement. There is no built-in way that I know of to place a Visual as a child of one of the standard elements (such as Border, Grid, Canvas, etc).  So we make our own:

{% highlight C# %}
Version 1
[ContentProperty("Child")]
public class VisualWrapper : FrameworkElement
{
    public Visual Child
    {
        get
        {
            return _child;
        }
 
        set
        {
            if (_child != null)
            {
                RemoveVisualChild(_child);
            }
 
            _child = value;
 
            if (_child != null)
            {
                AddVisualChild(_child);
            }
        }
    }
 
    protected override Visual GetVisualChild(int index)
    {
        if (_child != null && index == 0)
        {
            return _child;
        }
        else
        {
            throw new ArgumentOutOfRangeException("index");
        }
    }
 
    protected override int VisualChildrenCount
    {
        get
        {
            return _child != null ? 1 : 0;
        }
    }

    private Visual _child;
}
{% endhighlight %}

## Issue #2: Layout and the Loaded event

WPF provides a very convenient event called “Loaded”.  This event basically signals when an element has been fully initialized, measured, arranged, rendered, and plugged into a presentation source (such as a window).  Many elements use this event, including the MediaElement, but sadly this event is not raised for element trees that are not plugged into a presentation source, and displaying an element tree through the HostVisual/VisualTarget doesn’t count.  So work around this, we make our own presentation source and use it to root the element tree that the worker thread will own.  This immediately leads into another problem: layout is suspended on all elements until resumed by a presentation source.  Unfortunately the official mechanism to do this is internal, so the best we can do is to explicitly measure and arrange the root element.  Thus we have our VisualTargetPresentationSource class:

~~~ C#
public class VisualTargetPresentationSource : PresentationSource
{
    public VisualTargetPresentationSource(HostVisual hostVisual)
    {
        _visualTarget = new VisualTarget(hostVisual);
    }
 
    public override Visual RootVisual
    {
        get
        {
            return _visualTarget.RootVisual;
        }
 
        set
        {
            Visual oldRoot = _visualTarget.RootVisual;
 
            // Set the root visual of the VisualTarget.  This visual will
            // now be used to visually compose the scene.
            _visualTarget.RootVisual = value;
 
            // Tell the PresentationSource that the root visual has
            // changed.  This kicks off a bunch of stuff like the
            // Loaded event.
            RootChanged(oldRoot, value);
 
            // Kickoff layout...
            UIElement rootElement = value as UIElement;
            if (rootElement != null)
            {
                rootElement.Measure(new Size(Double.PositiveInfinity,
                                             Double.PositiveInfinity));
                rootElement.Arrange(new Rect(rootElement.DesiredSize));
            }
        }
    }
 
    protected override CompositionTarget GetCompositionTargetCore()
    {
        return _visualTarget;
    }
 
    public override bool IsDisposed
    {
        get
        {
            // We don't support disposing this object.
            return false;
        }
    }
 
    private VisualTarget _visualTarget;
}
~~~

## Background Threads

It is easy to make threads in C#.  One trick to be aware of is that you must mark the thread as being a “background” thread, otherwise the application will keep running as long as those threads are alive.  Also remember that parts of WPF require that its threads to be initialized for COM’s “Single Threaded Apartment”.   All of this is easy enough to do, and you’ll see this code later on:

``` C#
Thread thread = new Thread(/*…*/);

thread.ApartmentState = ApartmentState.STA;

thread.IsBackground = true;
thread.Start(/*…*/);
```

## The Demo

The demo will be a grid showing 3 movies, each rendered on a different background thread.  The demo is very simple, but hopefully gets the salient points across.  It is certainly possible to communicate from the UI thread to the worker threads, and given that all threads have their own dispatchers this is actually really easy, but this demo does not do that.  Interested readers should consult the documentation for Dispatcher.BeginInvoke.

## The XAML

The XAML simply defines a grid with 3 columns and puts a VisualWrapper in each column.  This is where we will be putting the HostVisual from code.  It would be cool  to build a class that automatically displays its content on another thread, but we would have to change the parser to support switching threads since the thread an object is created on is the thread that owns that object.  Anyways, the simple XAML is:

{% highlight XML %}
<Window x:Class="VisualTargetDemo.Window1"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:local="clr-namespace:VisualTargetDemo" 
    Title="VisualTargetDemo"
    SizeToContent="WidthAndHeight"
    Loaded="OnLoaded"
    >
  <Grid>
    <Grid.ColumnDefinitions>
      <ColumnDefinition Width="*"/>
      <ColumnDefinition Width="*"/>
      <ColumnDefinition Width="*"/>
    </Grid.ColumnDefinitions>
    <local:VisualWrapper Grid.Column="0"
                         Width="200" Height="100" x:Name="Player1"/>
    <local:VisualWrapper Grid.Column="1"
                         Width="200" Height="100" x:Name="Player2"/>
    <local:VisualWrapper Grid.Column="2"
                         Width="200" Height="100" x:Name="Player3"/>
  </Grid>
</Window>
{% endhighlight %}

## The Code

The code is also pretty simple.  For each of the 3 players, it creates a HostVisual on the UI thread, then spins up a background thread, creates a MediaElement, puts it inside a VisualTarget (which points back to the HostVisual), and puts it all inside our hacky VisualTargetPresentationSource.

{% highlight C# %}
public partial class Window1 : System.Windows.Window
{
    public Window1()
    {
        InitializeComponent();
    }
 
    private void OnLoaded(object sender, RoutedEventArgs e)
    {
        Player1.Child = CreateMediaElementOnWorkerThread();
        Player2.Child = CreateMediaElementOnWorkerThread();
        Player3.Child = CreateMediaElementOnWorkerThread();
    }
 
    private HostVisual CreateMediaElementOnWorkerThread()
    {
        // Create the HostVisual that will "contain" the VisualTarget
        // on the worker thread.
        HostVisual hostVisual = new HostVisual();
 
        // Spin up a worker thread, and pass it the HostVisual that it
        // should be part of.
        Thread thread = new Thread(new ParameterizedThreadStart(MediaWorkerThread));
        thread.ApartmentState = ApartmentState.STA;
        thread.IsBackground = true;
        thread.Start(hostVisual);
 
        // Wait for the worker thread to spin up and create the VisualTarget.
        s_event.WaitOne();
 
        return hostVisual;
    }
 
    private FrameworkElement CreateMediaElement()
    {
        // Create a MediaElement, and give it some video content.
        MediaElement mediaElement = new MediaElement();
        mediaElement.BeginInit();
        mediaElement.Source = new Uri("http://download.microsoft.com/download/2/C/4/2C433161-F56C-4BAB-BBC5-B8C6F240AFCC/SL_0410_448x256_300kb_2passCBR.wmv?amp;clcid=0x409");
        mediaElement.Width = 200;
        mediaElement.Height = 100;
        mediaElement.EndInit();
 
        return mediaElement;
    }
 
    private void MediaWorkerThread(object arg)
    {
        // Create the VisualTargetPresentationSource and then signal the
        // calling thread, so that it can continue without waiting for us.
        HostVisual hostVisual = (HostVisual)arg;
        VisualTargetPresentationSource visualTargetPS = new VisualTargetPresentationSource(hostVisual);
        s_event.Set();
 
        // Create a MediaElement and use it as the root visual for the
        // VisualTarget.
        visualTargetPS.RootVisual = CreateMediaElement();
 
        // Run a dispatcher for this worker thread.  This is the central
        // processing loop for WPF.
        System.Windows.Threading.Dispatcher.Run();
    }
 
    private static AutoResetEvent s_event = new AutoResetEvent(false);
}
{% endhighlight %}

## Summary

In this demo I’ve shown how to use the HostVisual and VisualTarget classes to compose pieces of UI from different threads.  There are some limitations: namely that the UI owned by the worker threads do not receive input events.  There were also some annoyances we had to work around along the way, but those proved to be fairly minimal.

The source code for this project is now included in my codeplex site: <http://www.codeplex.com/MicrosoftDwayneNeed>
