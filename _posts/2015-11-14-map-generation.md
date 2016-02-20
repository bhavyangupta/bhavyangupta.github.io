---
layout: post
title: Floor-plan annotation for CIS 573
---

For my CIS 573 final project we were extending an app that assists students/visitors in 
navigating around the campus. To that end I thought it is essential to have a 
"get directions" feature - much like Google maps. 

A problem with doing this is that how map annotation - i.e. how do you know which
room lies on which pixel coordinates and which regions on the map are 
traversable (like corridors)?

We annotated the room coordinates manually, but corridors were tricky. The following
video shows how I did it in MATLAB, followed by a brief explanation:

<iframe width="560" height="315" src="https://www.youtube.com/embed/H_oayX4lM3Y" frameborder="0" allowfullscreen></iframe>

The general idea is that the code ask the annotator to make a "traversable" 
trajectory on the floor-plan which is then converted to a undirected, closed and 
connected graph on which the planner can compute shortest paths.

The steps are as follows:

1. The user enters the full path of the target image.
2. The user enters key-points on the corridors that are essentially vertices of 
each polygonal section on the plan. The points are logged sequentially and the 
intermediate points are interpolated between successive points. This means that 
there have to be multiple key-points around some vertices when the floor-plan when
retracing the same region.
3. The points are linearly interpolated and the user asked for confirmation.
4. The graph formed is connected, but not closed. This means that the start/end
points are not the same and the planner won't be able to find the optimal path.
5. The multiple key-points at the retraced regions are clustered and the regions 
around them connected to the cluster center, effectively increasing the 
connectivity of the vertex node.

Finally, the points are stored as a JSON file for portability.
