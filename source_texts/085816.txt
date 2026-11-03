![noise_tor88_spheres_0](https://user-images.githubusercontent.com/39986899/126071555-ba04def2-03fd-4dab-8fe1-e6ccd085bf3e.jpg)

# About

  This project (t01) was a part of education.
  It created to do rendering of complex objects faster and easier on GPU with Ray Marching and Signed Distance Functions (SDF).
  ( Also if u want, there is part of Ray Tracing in 'shaders/rt'.
    Activate it by rebuilt project with prefix of shader 'rt' as directory in 'shaders/' )
    
# Short guide
<img src=https://user-images.githubusercontent.com/39986899/126071557-e383c54c-8096-426a-b6ad-6938dec50161.jpg width="80%">

  To build project u need to configure your platform to x86 in VS.

  Project can be rub with 't01.exe' and folder 'shaders'.
  In the 'shaders' can be found all shader, but u may be interested only in one file 'shaders/sdf/SDF.GLSL'
  In this file can be found some sort of script, that allows to create different virtual scenes.
  Realtime shaders files reloading, no need to restart the application.
  If u feel confident, can try configure scene right in 'FRAG.GLSL' in function 'sceneSDF' with varible 'mint' ( the closest object from Cam ).

# SDF description

Based on mathematical description of objects, we can determ the distance between object's surface and point in the space.

\< 0 - in object;

\= 0 - on the object's surface;

\> 0 - out of object;

<img src=https://user-images.githubusercontent.com/39986899/126071777-5bd5454d-f793-429a-994f-a0a9f8867c43.png width="30%">
<img src=https://user-images.githubusercontent.com/39986899/126071779-04c4f986-df4b-4ad8-aec0-053022c94064.png width="30%">
<img src=https://user-images.githubusercontent.com/39986899/126071781-538140cb-1a09-405c-acbc-30255d74819b.png width="30%">


**Operators:**
1.  Union (opU) - add new object to the scene ( objs may have intersections )
2.  Subtraction (opS) - cut from 1st object 2nd ( all points, that has 1st obj and don't have at the same time 1st and 2nd objs)
3. Intersection (opI) - all point that have 1st and 2nd objects at the same time
4. Smooth union (opUs) - add two new objects to the scene and makes a smooth conversion from one to the other obj depending on the coefficient 'k'

![tor88_spheres_box_smooth](https://user-images.githubusercontent.com/39986899/126071561-edd772b5-d3f1-45e1-b023-a2d600530ea0.jpg)

  # Script description

  - \#  - separator
  - @  - repeater -- @i\<number of repetitions>(\<function>)
  - '  - comment all string
  - // - comment all string
  - !  - embed all string

 
  **create the object:**
   -  \#\<position: vec3>#\<matrix>#\<object_function>()#\<material>
   
  **add to scene:**
  -   #\<function>\<obj>[\<obj>]
   
  **create material:**
  - #material{\<ambient>, \<diffuse>, \<specular>, \<gloss>}
 
# Examples

**Reflection and soft shadow**
![sin_plane_sphere](https://user-images.githubusercontent.com/39986899/126071559-d9868d9a-461d-401c-bc3e-de5f5be37faa.jpg)

**Plane with sinus modification and tor88 added with opUs**
![sin_plane_tor88_smooth](https://user-images.githubusercontent.com/39986899/126071560-7c2f047e-7c12-4ca9-9098-e95db5b3073d.jpg)

**Plane generated from noise; tor88, sphere and cube added by opUs; spheres**
![noise_tor88_spheres_1](https://user-images.githubusercontent.com/39986899/126071556-2b2f8be8-8659-408b-aba9-de1da93da3ad.jpg)

**Multiple reflection of spheres**
![repeat](https://user-images.githubusercontent.com/39986899/126071558-bea9865c-ddc9-4d85-a2c7-83a680c2c061.jpg)

**Tor with spheres addes by opS**
![tor_sphere_s](https://user-images.githubusercontent.com/39986899/126076900-4f866046-8f3b-432a-8e35-233aec201d0f.png)
