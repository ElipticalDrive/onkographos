# Gouraud_Shading

using C++ openGL to implement Gourand_Shading

command input files(.in) and .asc put in Debug

How to run this program?
---
use this format to run the project in cmd:

(cd to project folder)

projectname.exe inputfile.in

What's in input file(.in)?
---
(words behind ':' means input parameters)

first line means the size of openGL's window

1.scale: x y z

2.rotate: x y z

3.translate: x y z

4.observer: Ex Ey Ez COIx COIy COIz Tilt Hither Yon Hav

5.viewport: vxl vxr vyb vyt (=Vxmin Vxmax Vymin Vymax)

6.display

7.reset (initial Transformation Matrix)

8.end

9.#command

10.object: object.asc Or Og Ob Kd Ks N

11.ambient: KaIar KaIag KaIab

12.background: Br Bg Bb

13.light: index Ipr Ipg Ipb Ix Iy Iz


What's in .asc?
---
vertex_count plane_count

(coordinatation of vertices)

...

number_of_vertices (index of vertices) //start from 1, **clockwise**

...
