---
layout: barebones
title: Flipper your way into hardware - UI
date: 2025-05-21
permalink: /posts/flipper-into-hardware-ui/
series: Flipper into hardware
---

{% include series.html %}

{% include embed-audio.html src="/assets/audio/posts/flipper-into-hardware-p3/flipper-into-hardware-p3.mp3" title="Flipper your way into hardware p3" %}

At this point, we have a Flipper Zero application that steps a stepper motor. But, we want something that:

1. Looks pretty
2. Is configurable

So, we'll (try to) make that here.

# Requirements

Before, we were just exploring what it takes to write a Flipper Zero app that interacted with the pins - namely to step a stepper motor. Now we'll take that groundwork and use it to make a somewhat functional app. What could we want to do with our app?

1. Change the pins used to step and set direction of the motor
2. Start and stop the motor

There may be some other stuff, but I'm honestly quite lazy when it comes to this "UI" thing. We just want to configure things and make the motor function.

But first, we need a different UI.

# The Goal UI

The UI is a single view right now, but the Flipper Zero API gives a few more options. The first place that I looked at was [submenus](https://developer.flipper.net/flipperzero/doxygen/submenu_8h.html). These are just collections of options that you can press. Seems perfect, so we'll have one for each option:

> Goal UI
{:.filename}
``` c
+---------------+
| GO!           |
| Direction pin |
| Step pin      |
+---------------+
```

It's beautiful. Both of the pin options should be fairly easy since the official GPIO app has a [view that selects pins](https://github.com/flipperdevices/flipperzero-firmware/blob/dev/applications/main/gpio/views/gpio_test.c) - similar logic can be used here. That will definitely work out.

Then "GO!" will step the motor until back is pressed.

We will start with making this menu, then get selecting pins working.

## The submenu

We will have a submenu with multiple options to go to different "views." This is easily made with a [view dispatcher](https://developer.flipper.net/flipperzero/doxygen/view__dispatcher_8h.html) - this will hold all of the views and switch between them when needed. To set this up, I used the [view dispatcher example](https://github.com/flipperdevices/flipperzero-firmware/tree/dev/applications/examples/example_view_dispatcher) as a reference.

I won't go into every necessary part for code snippets, if you need completeness, simply [look at the code on Github](https://github.com/evantypanski/flipper-stepper).

First, we need to change the `StepperContext`. We removed the `ViewPort` and `FuriMessageQueue` and replaced them with the `ViewDispatcher`. Now the stepper context object (which is passed around a bunch of callbacks) is now this:

> stepper.c
{:.filename}
``` c
typedef struct {
  Gui *gui;
  ViewDispatcher *view_dispatcher;
  Submenu *submenu;
  Widget *stepping_widget;
  const GpioPin *dir_pin;
  const GpioPin *step_pin;
  bool state;
} StepperContext;
```

The `stepping_widget` will be used for the "GO!" view - our first view.

We need to change the `stepper_context_alloc` function to set up the view dispatcher and its views. To do that, remove the references to removed members (namely the `ViewPort` and `FuriMessageQueue`). Then, we'll start by creating the submenu:

> stepper.c
{:.filename}
``` c
  // Create and initialize the Submenu view.
  context->submenu = submenu_alloc();
  submenu_add_item(context->submenu, "GO!", SubmenuIndexStep,
                   stepper_dispatcher_app_submenu_callback, context);
```

There's a `SubmenuIndexStep` in there - that's actually an enum value defined at the top:

> stepper.c
{:.filename}
``` c
// Enumeration of submenu items.
typedef enum {
  SubmenuIndexStep,
} SubmenuIndex;
```

Since we're starting with just the step menu item, that enum only has one value, but we'll add more when we build the submenu up.

Next we'll create the "widget" that gets triggered when we switch to stepping mode:

> stepper.c
{:.filename}
``` c
  // Create the stepping widget.
  context->stepping_widget = widget_alloc();
  widget_add_string_multiline_element(context->stepping_widget, 64, 32,
                                      AlignCenter, AlignCenter, FontSecondary,
                                      "Stepping!");
  widget_add_button_element(context->stepping_widget, GuiButtonTypeCenter,
                            "Stop stepping", stepping_button_callback, context);
```

That's just a string and a button. Pressing the button makes it stop stepping.

Now we'll create the view dispatcher. This just manages the two views we just created:

> stepper.c
{:.filename}
``` c
  context->view_dispatcher = view_dispatcher_alloc();
  view_dispatcher_attach_to_gui(context->view_dispatcher, context->gui,
                                ViewDispatcherTypeFullscreen);
  view_dispatcher_add_view(context->view_dispatcher, ViewIndexSubmenu,
                           submenu_get_view(context->submenu));
  view_dispatcher_add_view(context->view_dispatcher, ViewIndexStepping,
                           widget_get_view(context->stepping_widget));
  view_dispatcher_set_custom_event_callback(context->view_dispatcher,
                                            stepper_event_callback);

  // Set the navigation, or back button callback. It will be called if the
  // user pressed the Back button and the event was not handled in the
  // currently displayed view.
  view_dispatcher_set_navigation_event_callback(context->view_dispatcher,
                                                navigation_callback);

  // The context will be passed to the callbacks as a parameter, so we have
  // access to our application object.
  view_dispatcher_set_event_callback_context(context->view_dispatcher, context);
```

The `ViewIndexSubmenu` and `ViewIndexStepping` are just part of an enum defined above:

> stepper.c
{:.filename}
``` c
typedef enum {
  ViewIndexSubmenu,
  ViewIndexStepping,
} ViewIndex;
```

The remaining parts are just adding those views and adding callbacks. The first callback is `stepper_event_callback` - we'll see this later. The `navigation_callback` is simpler - it's basically straight from the example. If we press back, and the app screen doesn't handle it, the navigation callback will exit. We'll see both of those soon, but first we have a couple more setup points.

This needs to be called from the "main" application code. After setting up the GPIO pins, we can "enter" the app easily with these calls:

> stepper.c
{:.filename}
``` c
  view_dispatcher_switch_to_view(context->view_dispatcher, ViewIndexSubmenu);
  view_dispatcher_run(context->view_dispatcher);
```

This will start at the submenu and run it. Then, we need to tear this down in `stepper_context_free`:

> stepper.c
{:.filename}
``` c
  view_dispatcher_remove_view(context->view_dispatcher, ViewIndexSubmenu);
  view_dispatcher_remove_view(context->view_dispatcher, ViewIndexStepping);
  view_dispatcher_free(context->view_dispatcher);
  submenu_free(context->submenu);
```

Okay, enough just rahashing the example. Now for new stuff.

### The Callbacks

So we have four callbacks that we assigned: 

1. `navigation_callback` - called when "back" is pressed

2. `stepper_dispatcher_app_submenu_callback` - called when a submenu button is pressed

3. `stepping_button_callback` - called when the "Stop stepping" button is pressed in the widget

4. `stepper_event_callback` - called by `stepper_dispatcher_app_submenu_callback` in order to start stepping

Now, it would be boring to show *all* of the code, so I'll just show the interesting parts in each.

In `navigation_callback`, we exit with `view_dispatcher_stop(context->view_dispatcher);` - then it will return to where it was called from:

> stepper.c
{:.filename}
``` c
  // navigation_callback

  // Back means exit the application, which can be done by stopping the
  // ViewDispatcher.
  view_dispatcher_stop(context->view_dispatcher);
```

`stepper_dispatcher_app_submenu_callback` actually triggers a different event conditionally, based on the submenu "index":

> stepper.c
{:.filename}
``` c
  // stepper_dispatcher_app_submenu_callback

  if (index == SubmenuIndexStep) {
    view_dispatcher_send_custom_event(context->view_dispatcher,
                                      ViewIndexStepping);
  }
```

Which triggers callback number 4 - we'll get to that.

`stepping_button_callback` has a couple of extra checks for a button getting pressed, provided by the example, in order to make sure it was a simple "press":

> stepper.c
{:.filename}
``` c
  // stepping_button_callback

  // Only request the view switch if the user short-presses the Center button.
  if (button_type == GuiButtonTypeCenter && input_type == InputTypeShort) {
    // Request switch to the Submenu view via the custom event queue.
    view_dispatcher_send_custom_event(context->view_dispatcher,
                                      ViewIndexSubmenu);
  }
```

Finally, the big one. `stepper_event_callback` actually has a few points worth mentioning.

### Back to Stepping

Here's the issue: if we just block like before, we no longer have the same loop structure with the event queue to check what events are handled. Instead, the view dispatcher handles events for us. So, once we enter, how would we exit? If we just naively loop, we won't leave.

We're already being somewhat imprecise about timing (namely by putting a lot of logic in the loop), so we aren't going to get the exact wait time that we want. Because of this, we have some freedom here to not reinvent anything. We'll do this with another callback!

The best reference I found for this is in the [Javascript documentation](https://developer.flipper.net/flipperzero/doxygen/js_event_loop.html#autotoc_md237). In that example, the Javascript code creates an event that fires a second after creation:

>
{:.filename}
``` js
// create an event source that will fire once 1 second after it has been created
let timer = eventLoop.timer("oneshot", 1000);

// subscribe a callback to the event source
eventLoop.subscribe(timer, function(_subscription, _item, eventLoop) {
    print("Hello, World!");
    eventLoop.stop();
}, eventLoop); // notice this extra argument. we'll come back to this later
```

We're doing something similar here, but in C. In our custom event, we want to set a timer that runs every `N` ticks. Since we don't really care about absolute precision here, we'll use `furi_event_loop_tick_set`. This can be done like so:

> stepper.c
{:.filename}
``` c
    furi_event_loop_tick_set(
        view_dispatcher_get_event_loop(context->view_dispatcher), 3,
        stepper_tick_callback, context);
```

And you can stop the timer in the same function with:

> stepper.c
{:.filename}
``` c
    furi_event_loop_tick_set(
        view_dispatcher_get_event_loop(context->view_dispatcher), 0, NULL,
        context);
```

Which one gets called depends on the `event` - if it's `ViewIndexStepping`, then we want to start stepping. If not, we want to stop stepping.

The callback `stepper_tick_callback` just ticks the motor once:

> stepper.c
{:.filename}
``` c
static void stepper_tick_callback(void *ctx) {
  furi_assert(ctx);
  StepperContext *context = ctx;
  furi_hal_gpio_write(context->step_pin, true);
  furi_delay_us(10);
  furi_hal_gpio_write(context->step_pin, false);
}
```

With all of this, we have... a button that says "GO!" - if you press it, it steps the motor, if not, it doesn't. Incredible.

### Choosing the Pin

Now, we want two more submenus, one for each pin (step and dir). Those are pretty easy, I just made an enum like this which will change the pin in each case in a callback:

> stepper.c
{:.filename}
``` c
// Each pin that can be used
typedef enum {
  PinA7,
  PinA6,
  PinA4,
  PinB3,
  PinB2,
  PinC3,
  PinC1,
  PinC0,
} UsablePins;
```

Okay, I avoided using the GPIO examples. It's simply annoying - if we started from the example, it would have been straightforward, but I like learning about this myself. Instead, we get more submenus, my favorite!

I won't actually go through the submenu process again, but we did it twice more: once for the step pin, once for the direction pin. These pins don't have extra validation - they may be the same. I don't want to know what happens if they are.

Here's what the app looked like after the two extra submenus:

![Home](/assets/img/posts/flipper-hardware-p3/home.jpg "Home screen"){: height="500em" }

![Pin select](/assets/img/posts/flipper-hardware-p3/pin-select.jpg "Pin select"){: height="500em" }

# UI Frustration

Developing for the Flipper Zero is enjoyable. I really liked using [micro Flipper Build Tool](https://github.com/flipperdevices/flipperzero-ufbt). GPIO worked as I would expect. But the UI development just stood in my way at every step. I wanted to do more - this was supposed to be a legitimate way for me to test stepper motors under various conditions! It's just tiring. I'm tired.

If you want to make a real application, go for it! I'm just not great at making anything pretty, so I ended up getting pretty frustrated. Feel free to try [Javascript](https://developer.flipper.net/flipperzero/doxygen/js.html) as well! It wasn't an option when I started this, and I know C better.

All in all, the Flipper Zero was fun to mess with. Making an application was rewarding. But, the UI just kept getting in my way. I don't think I'm cut out for screens.

Remember, the source code for all of this is [on my Github](https://github.com/evantypanski/flipper-stepper). Be warned, it should not be used as an example for how to do things right.
