---
layout: post
title: A 2023 pen plotter year in review
tags:
- Pen Plotter
- Python
---

For Christmas 2022 (yes, more than a year ago) I was given an iDraw v2 A3 pen plotter. I wish I could say I've used it for a year now, but the truth is that it sat largely unused for the first ten months of the year! Such is the life of a parent to young children.

This post is a summary of the two projects I've done in the last two months of the year. A Christmas card and a "snowflake machine" where I hooked it up to a joystick and let my nieces and nephews draw snowflakes.

I'd been hoping to use the pen plotter to create a Christmas card and so when November arrived I spent a chunk of time making that happen as my first serious project. I'm really pleased with the result, and it was a great way to learn how to use the plotter.

![Christmas card]({{ site.baseurl }}assets/2024-02-final-card.jpeg){:height="867px" width="650px"}

My final pipeline for creating the card was as follows:
 - Generate the card in py5 (a port of Processing to Python)
 - Export the different card layers as an SVG
 - Use vpype and vpype-gcode to stitch four cards together, convert the SVG to GCode files for each layer
 - Use gcode-cli to send the GCode to the plotter (via a connected raspberry pi so my mac wasn't permanently tied up)

It probably took me about 30 hours to muddle my way through the process and having done so, it laid the groundwork to be more adventurous with the snowflake machine. You can see the code for the Christmas card [here](https://github.com/sihil/christmas2023).

I came up with the idea for a snowflake machine when tidying up and coming across a long forgotten joystick. I wondered if I would use it to directly control the plotter but then also rotate and reflect what had been drawn to create a snowflake. The answer was no, you don't get snowflakes. But it did produce some interesting patterns and my nieces and nephews massively enjoyed playing with it.

The entirety of the snowflake machine was written in Python. I considered using vpype to do the rotation and reflection but in the end, I wrote my own code to do it, and it turned out to be surprisingly easy. The tricky part was getting the threading right so that the joystick input was independent of the control loop which was sending the GCode to the plotter.

The final structure of the snowflake machine was:
 - A thread to continuously read the joystick input that:
   - updated a state variable when the axes were changed that could be read by the control loop
   - run callbacks when buttons were pressed which made the control loop easier to write 
 - A control loop that:
   - monitored a z axis variable and sent pen up or down as required
   - read the x/y of the main stick, did some maths and sent GCode to the plotter to move the pen in the right direction
   - record the coordinates of each move
   - when the pen was lifted, rotate and reflect the coordinates and send the GCode to the plotter to draw the rotated and reflected lines

There is a lot of room for improvement but you can see the current state of the project [here](https://github.com/sihil/snowflake).

[//]: # (Initially I used Inkscape &#40;as the manufacturer supported package&#41; for sending SVGs to the plotter, but I quickly found that using the UI was annoying and searched for an alternative. I spent quite a lot of time trying to get saxi to work &#40;based on someone with an iDraw 1&#41; but it turns out that the iDraw 2 switched from the AxiDraw's EBB protocol to Gcode. )

