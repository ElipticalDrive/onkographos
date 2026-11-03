# Virtual Volume Ray Caster
VVRT (https://github.com/LukkeWal/VVRT, commit - 157ed3f56137dfb88922292a5bd0758dbcdf3420) has been extended with 'Level 16. ADS - Volume Raycasting' during BSc project "Visualisation of Acceleration Data Structures for Volume Raycasting in Virtual Ray Tracer"

Thesis abstract:

"As prevalent as acceleration data structures are within the field of computer graphics,
there remains a lack of suitable interactive applications developed to facilitate their
understanding. Virtual Ray Tracer (VRT), with its focus on interactivity and education, has addressed similar gaps since its conception, and, by building on the success
of its previous extensions, this thesis introduces another.

Through this thesis, VRT is extended to visualize acceleration data structures by highlighting their benefits in the process of volume raycasting. In particular, octrees and
the space-skipping techniques they facilitate are showcased with an emphasis on interactivity and educational value. Currently, VRT includes 15 tutorial levels covering
functionalities related to raytracing and volume raycasting. In this work, VRT is expanded with a new tutorial level that demonstrates octrees, their building process, and
the empty space skipping they enable within volume raycasting. The resulting extension aids in the education of students, as well as interested members of the general
public."

## Download
To try out a ready version of the VVRT, download the zip folder with [this dropbox link](https://www.dropbox.com/scl/fi/hwhk81eizhl7hnobqrh7m/VVRTBuild.zip?rlkey=wmn8j91fmpgfk30cylborq807&st=f1djuqp3&dl=0) which contians a build version for Windows. Extract the zip folder, open it and launch `VVRTBuild > Virtual Ray Tracer.exe`. The latest extension can be found in the Levels menu under the title "ADS - Volume Raycasting"

## Building the Application

As a prerequisite, you need a [Unity 2021.3.12f1 LTS](https://unity3d.com/unity/qa/lts-releases) release. 

- open the `UnityProject` folder with Unity [Unity 2021.3.12f1 LTS](https://unity3d.com/unity/qa/lts-releases)
- naviate to `CreateVoxelGrid > 3DTexture` and click the 3DTexture button to generate the 3D textures
- navigate to `File > Build Settings`, select your desired platform and press 'build'. The application has been tested on Windows
For more information on building Unity applications see the [Unity Manual page](https://docs.unity3d.com/Manual/BuildSettings.html).

## Future

The application is under active development and we hope that you will contribute to Virtual Ray Tracer, too; read on.

## License

The application is released under the MIT license. Therefore, you may use and modify the code as you see fit. If you use or build on the application we would appreciate it if you cited this repository and the Eurographics 2022 paper. As we are still working on the application ourselves, we would also like to hear about any improvements you may have made.

## Contact

Any questions, bug reports or suggestions can be created as an issue on this repository. Alternatively, please contact [Jiri Kosinka](http://www.cs.rug.nl/svcg/People/JiriKosinka).
