# vertex-and-fragment
This program demonstrates an implementation of both phong illumination, and notably fragment shading as well on different models. There is also directional light on one of the models.

Note: my contribution were fully implementing the shaders. The architecture of the code was given to me along with the models, canvas, and means of user interaction. My contributions are more specifically in the js/shaders folder.

To see the shaders working, open render.html in chrome. Then to experiment with light sources, use the various buttons on the top right to toggle modes, and then use the mouse to shift the lights around the space. 

The last two models on the right demonstrate the per-fragment shading I implemented. Per vertex shading is also there on the middle model and the one left of it (although the latter does not have a specular shading attribute).
