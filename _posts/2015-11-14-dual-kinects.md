---
layout: post
title: Using two kinects simultaneously
---

While working on PANDUbot, we wanted to use multiple kinects - one for mapping/
navigation and the other to assist in object detection. However, since the kinect
uses structured lighting, having multiple kinects look at the same scene causes 
interference in the two sensors, thereby causing "holes"  in the depth maps.

The solution to the problem (see references) turns out to be simple - you need 
relative motion between the two kinects so that the overlapping points in the 
structured lighting is minimised. A very simple way to do this is to attach a 
vibrating motor to one of the kinects that effectively produces tiny movement 
in the structured lighting, but is not so much as to cause motion blur in RGB 
images.

This effectively solves the problem. Here's a video of my results:

<iframe width="560" height="315" src="https://www.youtube.com/embed/sdPzqlj-LTI" frameborder="0" allowfullscreen></iframe>

Look for the following :

1. During the beginning, with the motor off, I cover one of the kinects and you 
can see that the depth image on the other kinect improves. **This demonstrates 
the interference** between the sensors.

2. At around 0:42, look for changes in the depth image - the holes in the floor 
suddenly disappear when I turn on the vibrating motor.


## References
http://research.microsoft.com/pubs/171706/shake'n'Sense.pdf