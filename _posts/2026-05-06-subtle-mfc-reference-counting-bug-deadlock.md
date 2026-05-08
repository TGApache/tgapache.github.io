---
title:  "How a Subtle MFC Reference Counting Bug Caused a Deadlock"
date: 2026-05-06
mathjax: false
layout: post
categories: media
tags: [cpp, mfc, legacy, bug, deadlock]
---

A hang is worse than a crash. There's no stack trace – just a live process that stops doing anything useful. Our application hung randomly after some period of time or user action. Developers spent half a week on it with no resolution. As the deadline approached, I was asked to take a separate look, and this is what I found.

## Some Background: MFC, DLLs, and Reference Counting

The application was built on MFC and structured
across multiple DLLs, some of which contained objects derived from `CWinApp` –
MFC's core application class. When you use MFC as a shared DLL, there's a rule
that matters enormously and is easy to miss: if you override `CWinApp::InitInstance()`
in your DLL, you *must* call the parent's `InitInstance()`. This is how MFC
tracks how many modules are actively using its global state. Every call to
`InitInstance()` increments an internal reference count. Every call to
`ExitInstance()` decrements it. When the count hits zero, MFC tears down its
globals.

Some of our legacy DLLs were not calling `super::InitInstance()` on load. They *were* calling `super::ExitInstance()` on unload.

So every time one of these DLLs unloaded, the reference count decremented – but
it had never been incremented in the first place. Eventually the count hit zero
prematurely. MFC decided it was time to clean up. Among the resources it cleaned
up were GDI+ resources – specifically things like `CPngImage` and `CImage`,
which we had recently started using for PNG images in the ribbon bar.

## Why PNGs Changed Everything

This is where it gets interesting.

We had recently migrated the application's UI assets from BMPs to PNGs (for `CMFCRibbonBar`). A
seemingly routine modernisation (yes, because nothing says 'modernization' quite like MFC). But PNGs involve GDI+, and GDI+ has a
behaviour that's critical to understand: **as soon as you start manipulating
images, GDI+ spins up a background worker thread**. It uses this thread
internally for image processing. It's not something you ask for. It just
happens.

This matters because of what GDI+ does on shutdown. `GdiplusShutdown` signals
that background worker thread and then *waits* for it to exit. The MSDN
documentation is explicit about this – and it includes a warning that's easy to
miss: _Do not call GdiplusStartup or GdiplusShutdown in DllMain or in any function that is called by DllMain._

## The Deadlock, Step by Step

Here's the sequence that produced the hang:

**1. A DLL unloads.**
Windows calls `DllMain` with `DLL_PROCESS_DETACH`. This runs inside the Windows
*loader lock* – a critical section that serialises all DLL load and unload
notifications. Nothing else related to DLL loading or unloading can happen while
you're inside it.

**2. MFC's reference count hits zero.**
Because `super::InitInstance()` was never called, the count was already lower
than it should have been. The unload call to `super::ExitInstance()` pushes it to
zero. MFC begins tearing down its global state.

**3. MFC invokes GDI+ shutdown.**
As part of that teardown, `GdiplusShutdown` is called. This happens inside
`DllMain` – inside the loader lock.

**4. GDI+ signals its background worker thread and waits.**
`GdiplusShutdown` tells the background thread to exit, then blocks, waiting for
it to finish. Our main thread is now waiting.

**5. The background worker thread tries to exit.**
To exit cleanly, it must send a `DLL_THREAD_DETACH` notification. To send that
notification, it must acquire the loader lock.

**6. The loader lock is already held.**
By our main thread. In step 1. Which is waiting for GDI+. Which is waiting for
the background thread. Which is waiting for the loader lock.

There's your deadlock.

```
Main thread (in DllMain / loader lock)
    └─ waiting for GdiplusShutdown
           └─ waiting for GDI+ worker thread to exit
                  └─ waiting to acquire loader lock
                         └─ held by main thread
                                └─ (deadlock)
```

It also explained why **Application-2** – a related product sharing much of the
same codebase and the same broken `InitInstance`/`ExitInstance` mismatch – never
exhibited the hang. Application-2 still used BMPs. No PNGs meant no GDI+
background thread. The same premature cleanup happened, but since GDI+ shutdown
was never invoked, there was nothing to deadlock against. The bug was latent,
waiting for something to activate it.

That something was the PNG migration.

## The Fix

Once the root cause was clear, the fix was straightforward. Every effected DLL that derived from `CWinApp` and overrode `InitInstance` needed to call
`super::InitInstance()` at the top of its override, matching the `super::ExitInstance()`
call it was already making on unload. This restored the reference count symmetry
MFC expected and ensured its global teardown – including GDI+ shutdown – only
happened when it was actually safe to do so. One call per DLL. That's the entirety of the code change.

```cpp
BOOL CMyDll::InitInstance()
{
    // This was missing. Without it, ExitInstance() would
    // decrement a count that was never incremented.
    CWinApp::InitInstance();

    // ... rest of initialisation
    return TRUE;
}
```

The PNG migration was the trigger, not the cause. The cause was a subtle API contract violation that had been sitting in the codebase, waiting for exactly the right conditions to break.

**Fix the source, not the symptom**. No more slapping `if (ptr != NULL)` and calling it done. Maybe I'll talk about that in a separate post.

_Takeaways? Some things should be left as an exercise to the reader._