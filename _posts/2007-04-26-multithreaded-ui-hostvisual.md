---
layout: post
title:  "Multithreaded UI: HostVisual"
date:   2007-04-26
categories: [WPF]
---

## Background: The WPF Threading Model

In general, objects in WPF can only be accessed from the thread that created them.  Sometimes this restriction is confused with the UI thread, but this is not true, and it is perfectly fine for objects to live on other threads.  But it is not generally possible to create an object on one thread, and access it from another.  In almost all cases this will result in an InvalidOperationException, stating that **“The calling thread cannot access this object because a different thread owns it.”**

## Freezables

Of course, there are exceptions to this restriction.  A familiar one is the Freezable class.  Freezable objects can be frozen, at which point they become read-only and we lift the single-thread restriction.  Examples of frozen Freezables include the standard brushes, available from the Brushes class.  These brushes can be used on any thread at any time.

This is incredibly useful, and used for all of the graphics primitive resources (pens, brushes, transforms, etc).  You can even derive your own types from Freezable and play by the same rules.
