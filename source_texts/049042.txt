# Distance_Estimator_render
Fortran implementation of a ray marching renderer that is feeded with a distance estimator field instead of a polygon mesh.

Distance Estimator and the renderer uppon it are implemented as proposed by Inigo Quilez in his webpage http://www.iquilezles.org

Files content:

1.- main.f95
  It strides the output matrix dtermining for each pixel on the screen the accurate versor for the ray marching to use. Once the versor is set the "combo" subroutine is called to get final RGB values. When "subSample" differes from 1, many versors are computed for the same pixel and then averaged. Finally the output is saved to a BMP file by means of the "saveBMP" subroutine.
  
2.- DEparameters.f95
  Holds the "DEparam" module, where every needed parameter is stated.
  
3.- DErender4.f95
  Contains the various subroutines which take care of every action implemented for the image rendering. Mainly the "combo" subroutine, which calls the first "trace" and "colour" subroutines that get the hit-point coordinates and normal to the surface and the RGB values of the hitted object. Later "reflection" subroutine is called to take care of the reflected ray marching.

4.- DE.f95
  Implements the only function "DE" in the "DistanceEstimator" module. This function returns an scalar for every possible point in the space. That scalar is the distance from that point to the nearest object in the scenario that needs to be rendered. The "trace" subroutine uses this distance as a safe step forward along the given ray marching direction and calls the "DE" function again, until the returned distance is smaller than the "minD" parameter.
  Also, when "pixel" argument provided, "DE" returns the RGB values of the hitted surface.

5.- bmp.f95
  This module contains the "saveBMP" subroutine used to output the "out" array in an easy-to-watch way.
