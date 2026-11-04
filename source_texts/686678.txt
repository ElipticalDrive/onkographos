# Ray Tracer With Deep Shadow Map

Deep shadow maps is an efficient shadow rendering technique, especially useful for producing
fast and high-quality shadows for primitives such as hair, fur, and volumetric objects. In comparison to traditional shadow maps, deep shadow maps are smaller and significantly faster to
access, making them ideal for achieving high-quality shadows. Additionally, unlike traditional
shadow maps, they support partially transparent surfaces, volumetric objects, and motion blur

During the project for my graphics programming class I implemented the Deep shadow map technique to my Ray Tracer

# How to run the project
This project is purely written in c++ and does not contain external dependencies. You can
execute it directly as a c++ project. The project folder is ”DeepShadowMaps”. When the
project is executed, the program will create a ppm file named ”render.ppm” that contains the
final render.

# Test Scences
All of the scenes from the abstract or the presentation are on the SceneSetUp.cpp file. You can
modify this file to change the content of the scenes. To render a specific scene you just have
to call it from the main.cpp file. Or you can uncomment one of the following files still in the
main.cpp file.

![Image to render diffrent scenes](https://github.com/OnurBasci/Ray-Tracer-Deep-Shadow-Map/blob/master/Docs/images%20readme/Capture%20d'%C3%A9cran%202024-05-06%20120124.png)

# Parameters
From the main.cpp file you can change the Ray Tracing and shadow map resolution with it’s
sample number. You can also choose the shadow type between classical ray tracing shadows
and deep shadow map shadows. There is also an option to activate self shadowing.
To change the volumetric objects appearance you can modify the scaling factor and noise
power parameters from MathHelper.cpp file. You can also give different density functions from
this file as a material for volumetric objects.
