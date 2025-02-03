---
layout: barebones
title: Flipper your way into hardware - Stepping
date: 2025-02-03
permalink: /posts/flipper-into-hardware-step/
series: Flipper into hardware
---

{% include series.html incomplete=true %}

{% include embed-audio.html src="/assets/audio/posts/flipper-into-hardware-p2/flipper-into-hardware-p2.mp3" title="Flipper your way into hardware p2" %}

Last time we left with a (crappy) app that would turn an LED on when you press the OK button and back off when you press it again. Now, we'll take that and turn it into a stepper motor app - which is just the same thing but with a couple more moving parts (literally :) ).

# Stepper motors

So, first: How do stepper motors work? Well, stepper motors are a kind of brushless DC electric motor with discrete steps and don't need a position sensor... well, that's kind of secondary. I guess the best answer is: magnets!

For our purposes, we mainly care about how to make the stepper motor spin in circles (because that's *definitely* useful). When you program a microcontroller to control a stepper motor, you'll use a stepper motor driver (which itself is a microcontroller) - I will be using the [A4988](https://www.pololu.com/product/1182).

There are tons of tutorials on how to program these with an Arduino (I used [this one](https://www.makerguides.com/a4988-stepper-motor-driver-arduino-tutorial/) a couple times). I'll skip over wiring the motor and an external power supply and all of that, see some of those for specifics. What I care about is: how can I make a Flipper spin this?

For that, all we care about are two pins on the driver: `DIR` and `STEP` - one to change the direction we spin and one to step the motor. It's really simple, we just send some pulse to `STEP` whenever we want to do a "step" on the motor - which may be smaller or bigger increments depending on if you wired with microstepping.

All of that to say: We need two pins. That's it. One for direction, one for stepping. Let's try it.

# Adding a pin

This is really easy - first we just get separate pins for `dir` and `step` in the `StepperContext` object:

> stepper.c
{:.filename}
``` c
typedef struct {
    // ...
    const GpioPin *dirPin;
    const GpioPin *stepPin;
} StepperContext;
```

Then we'll just go back and add a new parameter to our setup function:

> stepper.c
{:.filename}
``` c
static StepperContext* stepper_context_alloc(uint8_t dirPin, uint8_t stepPin) {
```

This sets them up in the same way as before, but with different names:

> stepper.c
{:.filename}
``` c
  context->dirPin = furi_hal_resources_pin_by_number(dirPin)->pin;
  context->stepPin = furi_hal_resources_pin_by_number(stepPin)->pin;
```

Then we call this from the main `stepper_app` function along with setting up the pin:

> stepper.c
{:.filename}
``` c
  StepperContext *context = stepper_context_alloc(6, 7);
  furi_hal_gpio_write(context->stepPin, false);
  furi_hal_gpio_init(context->stepPin,  GpioModeOutputPushPull, GpioPullNo, GpioSpeedLow);
  furi_hal_gpio_write(context->dirPin, false);
  furi_hal_gpio_init(context->dirPin,  GpioModeOutputPushPull, GpioPullNo, GpioSpeedLow);
```

And finally reset the pins at the end:

> stepper.c
{:.filename}
``` c
  furi_hal_gpio_init(context->stepPin, GpioModeAnalog, GpioPullNo, GpioSpeedLow);
  furi_hal_gpio_init(context->dirPin, GpioModeAnalog, GpioPullNo, GpioSpeedLow);
```

(obviously, changing `pin`->`stepPin` where it was used before).

Surprisingly, if you wired it right, this will do something! It will run one step every time you create a "rising edge" - that is, pressing the OK button to go from low to high. You can see this if you put tape or something on your stepper motor. You'll also probably hear it.

Cool - now let's make it a little more usable.

## Toggling continuous steps

First, it'd be nice to have this toggle continuous steps rather than just toggling a single step. To do this, we will repurpose the `state` variable in the context: now it'll tell us if we're stepping or not. That means we don't need to change much to get this working - it'll still trigger with OK and no more or fewer variables!

We do need to change the time out for `furi_message_queue_get` - we had it set to `FuriWaitForever` which would make it so we step once... then wait for input. Instead, we can wait forever if we're not stepping, but when stepping we want to continue without waiting. We do this like so:

> stepper.c
{:.filename}
``` c
    // If we're stepping, don't wait
    uint32_t timeout = context->state ? 0 : FuriWaitForever;
    const FuriStatus status = furi_message_queue_get(context->event_queue, &event, timeout);
```

Surprisingly enough, there is exactly one more thing that needs done: stepping! That's just turning the `STEP` pin high then low with a delay between, which I do at the top of the loop so that it's always run:

> stepper.c
{:.filename}
``` c
    // Step before checking anything else
    if (context->state) {
      furi_hal_gpio_write(context->stepPin, true);
      furi_delay_us(10);
      furi_hal_gpio_write(context->stepPin, false);
      furi_delay_us(3000);
    }
```

The time high (after writing `true`) is inconsequential here since we really only care about the rising edge. Calling the function itself has overhead. We'll wait 10 microseconds for the heck of it.

Then we wait a bit of time after stepping. I was a bit generous and gave it 3 milliseconds, but you can figure it out for your motor and driver. I'm not making anything where that would matter.

Now, the keen eyed among you will remember that when setting up the pins, we passed in `GpioSpeedLow` - but now we're waiting a very small amount of time!

> stepper.c
{:.filename}
``` c
  furi_hal_gpio_init(context->stepPin, GpioModeAnalog, GpioPullNo, GpioSpeedLow);
```

Is that okay?

Well, the short answer is, "I think so." Basically, you want to choose the lowest speed that works for your context. I get a spinning motor with this, so obviously it works! But, if you want more information, that's actually a product of some registers in the underlying microcontroller, the [STM32](https://www.st.com/en/microcontrollers-microprocessors/stm32-32-bit-arm-cortex-mcus.html). If you're curious, there's a whole [194 page datasheet](https://www.st.com/en/microcontrollers-microprocessors/stm32wb55rg.html) you can rummage through.

# The app so far

So now we have a real application which will toggle spinning a motor:

![Stepping](/assets/img/posts/flipper-hardware-p2/stepping.gif "Stepping that motor"){: height="500em" }

It's pretty cool that in 2 short posts we're at a place where something physical is happening from the Flipper. There's a lot more that can be done here to make the app useful and pretty.

For now, though, this was my intention when I set out to make this app: Interacting with hardware via the Flipper Zero. There's pretty active development on the Github for ways to make the app-building experience easier (like [Javascript](https://www.reddit.com/r/flipperzero/comments/1h5lsuu/introducing_the_javascript_sdk_for_flipper_zero/)).

Until then, enjoy... flipping? Stepping?
