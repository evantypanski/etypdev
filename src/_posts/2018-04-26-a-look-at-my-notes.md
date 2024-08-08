---
layout: barebones
title: A Look At My Notes
date: 2018-04-26
permalink: /posts/a-look-at-my-notes/
---

Notetaking is a hotly debated topic, with [many saying](https://www.npr.org/2016/04/17/474525392/attention-students-put-your-laptops-away) that taking notes by hand is simply better than digitally.  This seems to be because the actual process of transcription tends to be more important than rereading the notes.  I don't find that all that surprising - I hardly read back my notes simply because it doesn't seem to benefit me.  I believe many (but not all) feel the same.

The actual process of writing is far slower than typing.  I'll use myself as an example - I tested myself using [this paragraph](https://10fastfingers.com/text/30765-Simple-Paragraph) from a random typing speed site and made entirely sure not to try more than once.  When I wrote the paragraph by hand, it took 2 minutes 20 seconds.  Typing it with a fair number of corrections (21 according to the site) took 55 seconds, clocking at 79 words per minute, which is slower than normal.  The takeaway?  My typing speed is over twice as fast as my writing speed.

That doesn't even account for the messiness of my handwriting.  I write like a five year old - my base line is all over the place and I need to look at the page.  I clench the pencil too hard so it hurts.  Typing, on the other hand, is always neat and comfortable.  Since switching to Dvorak keyboard layout ([see my other blog post](https://www.etyp.dev/posts/on-alternative-keyboard-layouts)), I have found that I am much more comfortable typing than with QWERTY.  Basically, I get incredibly tired writing, but not typing.

We can keep going on how much better typing is because you can relay more information in a neater format.  But, what's the benefit of writing?  I said previously that I don't write my notes to reread them.  I write notes to retain information the second I learn it.  [Studies show](http://drawingchildrenintoreading.com/assets/the_pen_is_mightier_than_the_keyboard-libre.pdf) that writing notes simply lets you retain more information.  I would likely agree with that finding for most individuals.  The primary reasons seem not to be a product of the laptop, but instead a product of how the individuals use the laptop.

## How I Take Notes

A short history, I learned LaTeX in Fall 2016 in order to take notes and do homeworks more efficiently in my discrete math class.  This was the beginning of my first year at college.  Before I learned LaTeX, I took notes using my laptop, but generally in simple Microsoft products.  This was incredibly cumbersome for equations and prettiness.  I downloaded [MikTex](https://miktex.org/) for Windows and struggled through the first month of not knowing how to do practically anything.  By the end of my first year, most of my notes were typed with LaTeX and generally were pretty ok.

Here's an example of some discrete math that I took my first year.  These notes were often rewritten to make more clear as well (part of how I study):

![Discrete Math](/assets/img/posts/a-look-at-my-notes/discrete.png "Discrete Math")

I'm definitely not going to claim these are perfect notes, but it certainly provided a good reference as well as forcing me to write down what each law was.  I likely did this after the class that we learned it, but this was very common for my notes: to have every piece of information.

My more recent notes are different based on the class.  For example, in algorithms, pictures are often very useful, but I'm not about to draw a flow graph and scan it into my notes.  Thus, I just describe it.  The key is, however, I ensure that I learn the content and then write my translation of the lecture.  It's similar to what you have to do when hand writing notes, but I do it voluntarily while typing.  The following is an example of the result:

![Algo](/assets/img/posts/a-look-at-my-notes/algo.png "Algorithms")

These notes seem obviously less curated than my discrete math notes - I take that as a sign of maturity.  I've learned that I can't just write everything down.  I could make a table with definitions and have a huge professional notebook that can essentially also be a textbook.  But, I'm not going to look back at these notes that much, so why would I?  I might ensure that my intuition now matches my intuition when I first learned the concepts, but other than that these notes will likely just have their use from my writing them in the first place.  I don't need to make it awesome and pretty - I need to make it easy to do what I need to do.

The main increase in benefit occured when I switched from using MikTex to using Vim on a Linux environment.  The main reason is that Vim helps me write notes even more efficiently, should I need to change something.  This created a problem because I didn't know how to create a PDF efficiently other than just compiling the notes with `pdflatex algo.tex`.  This was fixed when I found [Vim Latex Live Preview](https://github.com/xuhdev/vim-latex-live-preview), a Vim plugin that recompiles the PDF document as you type (after a certain amount of inactivity at least).  It doesn't use a ton of computing power and it feels very natural.  This has been what's changed my notetaking from being pretty to being efficient as well.  Combine this with a tiling window manager and this setup is perfect for ensuring your notes serve their purpose, compile correctly, and look how you want them to look.

I won't go too much into how I organize sections and lecture dates, mostly because that's unimportant.  It's loosely correlated with the textbook in most cases.  But, I will say that taking my notes in LaTeX has made it more likely for me to look back at my notes, as I feel more comfortable grabbing an equation from my notes rather than lecture slides.  This gives me the best of both worlds: so long as I do not get distracted during class and I make an effort to write the notes in my own words, I've found this works for me.  It might not work for everyone, but it definitely works for me.


