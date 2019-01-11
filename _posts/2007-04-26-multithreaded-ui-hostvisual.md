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

## Issue 1: Hosting a Visual in XAML

The first problem to solve is that the HostVisual class derives from Visual.  I can't use an existing panel, such as Border, to host this visual.  Border derives from Decorator, which is the standard base class for panels that have a single child.  Unfortunately, the child is strongly typed to be a UIElement.  I have to use a HostVisual, which does not derive from UIElement. There is no built-in way that I know of to place a Visual as a child of one of the standard elements (such as Border, Grid, Canvas, etc).  So we make our own:

~~~ ruby
def what?
  42
end
~~~
