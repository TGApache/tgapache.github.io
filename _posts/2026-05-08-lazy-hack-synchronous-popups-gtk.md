---
title:  "A Lazy Hack for Synchronous Popups in GTK"
date: 2026-05-08
mathjax: false
layout: post
categories: media
tags: [win32, gtk, c, ui]
---

If you've ever worked across both Win32 and GTK, you would know they don't always agree on how things should work. This is one of those moments.

---

## The Problem

On Win32, `TrackPopupMenu` is synchronous. You call it, the menu appears, the
user makes a selection, and the function returns. Simple, predictable, easy to
reason about.

GTK doesn't work that way. `gtk_menu_popup_at_pointer` fires and returns
immediately — the menu appears asynchronously and your code keeps running. If
you're porting something from Win32, or just want that same "block until done"
behaviour, GTK gives you nothing out of the box.

---

## The Hack

The comment in the code says it best: *"a lazy hack"*. But sometimes a lazy hack
is the right call.

```cpp
static void
tray_track_popup_menu_at_pointer(GtkMenuShell* popup_menu)
{
  volatile bool selection_done = false;

  static auto menu_item_selection_done_cb = [](GtkMenuShell *Menu, gpointer user_data) -> void
  {
    volatile bool* selection_done = (volatile bool*) user_data;
    *selection_done = true;
  };

  gulong handler_id = g_signal_connect(G_OBJECT(popup_menu), "selection-done",
                  G_CALLBACK((MenuItemSelelctionDoneCb)menu_item_selection_done_cb),
                  (void*)&selection_done);

  gtk_menu_popup_at_pointer(GTK_MENU(popup_menu), NULL);

  while(selection_done == false)
  {
    main_loop_iteration(/*blocking*/1);
  }

  g_signal_handler_disconnect(G_OBJECT(popup_menu), handler_id);
}
```

The idea is straightforward. A `volatile bool` flag starts as `false`. We connect
a callback to GTK's `selection-done` signal — which fires when the user picks
something or dismisses the menu — and that callback flips the flag to `true`.
Then we spin on the main loop, processing events one blocking iteration at a
time, until the flag flips. Once it does, we disconnect the signal handler and
return.

To the caller, this function looks and feels synchronous — exactly like
`TrackPopupMenu`.

---

## Why `volatile`?

The `volatile` keyword here is doing real work, not just decoration. Without it,
the compiler might see a bool that never changes within the loop body and optimise
the check away entirely — effectively turning the while loop into an infinite loop
or eliminating it. `volatile` tells the compiler: *don't cache this, check it
every time, it can change from somewhere you're not tracking.*

---

*Is It Pretty? No. Does It Work? Yes.*