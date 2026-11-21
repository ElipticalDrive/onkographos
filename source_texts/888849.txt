# RayGraphics

This project implements ray tracing and ray marching algorithms to render simple geometric shapes like spheres, cylinders, 
boxes and cones. To use it, you first need to create a Scene object and then add objects and light sources (see Scenes.scala). 
Then you can use Scene.getPixelColor(Int, Int) method, which returns a Color object for given pixel coordinates.
An example of displaying the image in a new window is given in Display.scala, which contains the main function.
