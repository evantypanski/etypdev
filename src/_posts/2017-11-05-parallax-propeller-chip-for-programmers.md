---
layout: post
title: Parallax Propeller Chip for Programmers
date: 2017-11-05
permalink: /posts/parallax-propeller-chip-for-programers/
---

Recently, I taught a workshop on basic microcontrollers for those who have little to no experience programming.  No one who attended had even seen C code outside of trivial examples.  However, with microcontrollers being extremely useful for the projects I do in Gizmologists (a club at the University of Virginia).  Now, I want to appeal to those who already understand how to program.

Note, I will only go into the semantics of how to program the Propeller chip and the fundamental knowledge behind it.  I will not cover any circuits or wiring that may occur as there are many tutorials for microcontrollers like Arduino which cover this material - it's exactly the same here.  Also, I'm assuming for the sake of this tutorial that those reading it have some knowledge of programming.  My goal is to teach how the Propeller chip achieves its goals within mechatronics.

## The Propeller Chip

The [Parallax Propeller chip](https://www.parallax.com/sites/default/files/downloads/P8X32A-Propeller-Datasheet-v1.4.0_0.pdf) is a very unique microcontroller - unlike most, who resort to interrupts and multi-threading for solutions, the propeller chip actually uses parallel processing.  This is incredibly rare in the microcontroller world - you almost never get a chip that has decent processing power (20 million instructions per second running machine code) on each of eight parallel processors.

The Propeller chip has two main languages which you may use: Spin, which is a high level interpreted language, or C, much like many other microcontrollers.  Since Spin was made for the Propeller chip, the language has its own quirks in order to make programming the Propeller chip easier.  For example, even in a high level interpreted language, register manipulation can be done with a simple array index: `dira[0]~` is equivalent to setting the 0th bit of the dira register equal to 0.  Conversely, `dira[0]~~` is equivalent to setting the 0th bit equal to 1.  This syntax is equivalent to `dira[0] := 0` or `dira[0] := 1` in Spin.

Registers are, in general, special hardware locations that are used to store small pieces of data, likely a memory address.  Sometimes, registers may contain special information: for example, the program counter contains the address of the current instruction.  The registers on microcontrollers often have a more useful purpose for day to day programming: often times, one can set the state of a GPIO (general purpose input output) pin by directly changing a register's stored value.  This is an incredibly quick operation, as such it is perfect for manipulating hardware from microcontrollers.

Thus, registers are incredibly important for the Propeller chip.  The most important registers are `dira`, `outa`, and `ina`.  The `dira` register is used to set whether a pin is input (0) or output (1).  The array index should be the pin number that is being assigned.  The `outa` register is used to set the output state of a pin - either 0 (off) or 1 (on).  Finally, the `ina` register is used to read the input, either a 0 or 1, from a certain pin.  This is most likely a button or switch that simply closes a circuit.

These fundamentals are not as important once we go into the C code, but it is useful to understand what goes on behind the scenes.  Think about each pin as a switch: the `dira` register tells the Propeller chip whether it will be reading in some input value or outputting some current.  Then, when `outa` is set to 1 for that pin, it sends a current through the circuit.  When `outa` is set to 0, it stops the current from flowing through that pin.  Likewise, when `ina` is accessed, the Propeller chip will read whether it sees a voltage or not.  If it does, then `ina` will read a 1 for that pin, else it will read a 0.

## Propeller C

While Spin is the original language of the Propeller chip, it is often more useful to use and teach C, as more people know C and it's more universal in the world of embedded systems.  Thus, despite C being much like an afterthought for the Propeller chip, I will dive most into C programming on the Propeller chip.

Should you want to follow along, the Propeller C IDE, named SimpleIDE, can be installed using [this tutorial](http://learn.parallax.com/tutorials/language/propeller-c/propeller-c-set-simpleide).

The major difference between C and Spin is that C abstracts much of the hardware interaction.  For example, instead of directly accessing the `dira` register to set a pin's direction, one sets the direction of a pin with a call to the function `set_direction(int pin, int direction)`.  For example, `dira[0]~~`, which sets pin 0 to output (1), is equivalent to `set_direction(0, 1)` in C.  

``` C
// set_direction function

void set_direction(int pin, int direction)
{
    unsigned int mask = 1 << pin;
    if (direction == 0)
    {
        DIRA &= (~mask);
    }
    else
    {
        DIRA |= mask;
    }
}
```

Looking at the source code for this function, we can see that it simply manipulates the `dira` register.  However, it makes sense that this would be abstracted - manipulating a register in C is slightly more involved, as one must bitshift the desired value and use bitwise AND (set to input) or bitwise OR (set to output) to set the correct pin.  For example, this is how one would set pin 2 to output by directly manipulating the `dira` register in C: `DIRA |= (1 << 2);`  

Likewise, manipulating outputs is abstracted.  The functions `high(int pin)` and `low(int pin)` are used to make a certain pin high (send a current through it) or low (stop the current).  Once again, these functions both just manipulate the `outa` register in a similar fashion to the example above.

Finally, reading inputs can be done with the `input(int pin)` function.

With the fundamental knowledge of how registers can manipulate pins, let's look at a simple example of a stoplight simulation, assuming green is on pin 0, yellow on pin 1, and red on pin 2.

``` C
#include "simpletools.h"        // Simpletools contains a lot of useful functions

void main() {
    set_direction(0, 1);        // Sets pin 0 to output
    set_direction(1, 1);        // Sets pin 1 to output
    set_direction(2, 1);        // Sets pin 2 to output
    while (1) {                 // Infinite loop
        high(0);                // Sets pin 0 to high
        pause(1000);            // Pauses one second
        low(0);                 // Sets pin 0 to low
        high(1);                // Sets pin 1 to high
        pause(750);             // Pauses 0.75 seconds
        low(1);                 // Sets pin 1 to low
        high(2);                // Sets pin 2 to high
        pause(1000);            // Pauses one second
        low(2);                 // Sets pin 2 to low
    }
}
```

Now we can see that the world of microcontrollers isn't really a scary place with low level hardware manipulation - in fact, much of the hardware interaction is abstracted behind functions.  All that needs to be known is the pins that LEDs are connected to and a simple stoplight should be trivial.

## Parallel Processing

Now that we have fundamentals of microcontrollers down, let's take a look at how the Propeller chip handles multiple processes at once.

The Propeller chip is unique in one primary way: it has eight parallel processors, called cogs.  Each can store 2 KB of RAM, stored in 512 longs, with each one being synchronized with the main system clock.  Should more information be desired, look at [this datasheet](https://www.parallax.com/sites/default/files/downloads/P8X32A-Propeller-Datasheet-v1.4.0_0.pdf).  Other microcontrollers, such as Arduino, focus on multithreading (or, in the case of Arduino, protothreading - [see this post](https://create.arduino.cc/projecthub/reanimationxp/how-to-multithread-an-arduino-protothreading-tutorial-dd2c37) for how it really works).  This, however, only simulates multiple processes through concurrency.  The Propeller chip allows us to run a stoplight, turn a motor, constantly check for certain switches being pressed, and check if the time has changed, all without interference.

Furthermore, it's incredibly simple to program.  Here's an example from the workshop I gave:

``` C
#include "simpletools.h"        // Simpletools contains a lot of useful functions

void blinkLED1();               // Forward declaration 

unsigned int stack[100];        // Stack in shared memory as user defined stack space

int main() {
    cogstart(&blinkLED1, NULL, &stack, sizeof(stack));    // Starts new cog process
    set_direction(0, 1);        // Sets pin 0 to output
    while(1) {                  // Infinite loop
        high(0);                // Sets pin 0 to high
        pause(500);             // Pauses 0.5 seconds
        low(0);                 // Sets pin 0 to low
        pause(500);             // Pauses 0.5 seconds
    }
}

void blinkLED1() {
    set_direction(1, 1);        // Sets pin 1 to output
    while(1) {                  // Infinite loop
        high(1);                // Sets pin 1 to high
        pause(750);             // Pauses 0.75 seconds
        low(1);                 // Sets pin 1 to low
        pause(750);             // Pauses 0.75 seconds
    }
}
```

This is meant to visually show how parallel processing is achieved: should there be an LED in pin 0 and 1 with this code, they will blink at offset intervals from each other.  Again, this is trivial, but very important.

The stack used for cogstart is not extremely important - unless space is needed, I would suggest just putting 100 unsigned ints until that seems insufficient.  The minimum amount needed is 40 - the more the merrier, though.

### Locks
The Propeller chip allows each cog to access central hub RAM, where global variables may be pulled and manipulated.  This can create problems, especially if another cog needs to use that variable.  The Propeller chip has a relatively simple implementation of semaphores (or locks) in order to prevent the danger of race conditions and the like.

The Propeller chip has eight locks, of which seven are usable.  Locks are shared by the entire system, so each cog can check to see if a lock is being used before manipulating any shared data.  This is done with the following:

The function `locknew()` is first used to get a lock ID.  This should be assigned to some variable which all cogs that might access the data have access to.  Then, `lockset(int lockid)` will set the lock.  If the lock is already set, this function will wait until the lock is cleared before setting the lock, and return true on success.  Then, once manipulation of the data is complete, you can simply call `lockclr(int lockid)` in order to clear the lock and allow others to use it.  If the lock is not going to be used after a certain point, it is wise to use `lockret(int lockid)` to return the lock to the pool so that others can get it via the `locknew()` function.

## Diving Further
Since Parallax has focused on education, it seems that most of the Propeller tutorials are aimed at the younger generation.  This abstracts much of what we may want to learn about the Propeller chip, as it truly is a fantastic chip for integration with motors and sensors.  Thus, it is wise to understand how to read the source code for the provided C libraries.

The libraries have decent documentation, found [here](https://cdn.rawgit.com/parallaxinc/propsideworkspace/master/Learn/Simple%20Libraries%20Index.html).  You can also go into the Learn directory, which should be in the Simple IDE directory which stores your projects (this is a default from SimpleIDE).  The HTML documentation found inside these directories is quite good, but you can also look at the source code to see what goes on behind the scenes - maybe make improvements on it for efficiency increase as well.

Also, the Propeller chip has a lot more hardware components that I didn't even mention in this post - how the central Hub RAM interacts with the cogs, different control registers, and much more.  In order to understand all of this, it would be wise to give the [propeller datasheet](https://www.parallax.com/sites/default/files/downloads/P8X32A-Propeller-Datasheet-v1.4.0_0.pdf) a read - this helps in understanding what sort of magic goes on behind the scenes.

## Conclusions
The Parallax Propeller chip is an incredibly useful chip with parallel processing capabilities that just aren't present in other microcontrollers.  This makes the Propeller chip especially useful in almost all applications of microcontrollers.

While I didn't provide any explanation on the wiring of pins and LEDs, it is quite easy to get a feel for how this works with the tutorials that are out there.  Parallax has [decent tutorials](http://learn.parallax.com/tutorials/propeller-c) for learning this.  I wanted to focus on how the Propeller chip does what it does rather than explain how to make the Propeller chip achieve certain goals.  In other words, most tutorials show you what to do to achieve a goal; I wanted to show how the Propeller chip achieves these goals.

Also, these situations might seem very trivial.  However, in most applications of mechatronics, most of what you're doing is simply manipulating when the pins provide an output or take an input.  With knowledge of how a specific device works, such as a stepper motor or analog digital converter, it is quite easy to get what you want only with what I've explained.

