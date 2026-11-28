# 3D Reconstruction Using Accelerated Moving Least Squares and Marching Tetrahdra

This is a minimal dependency implementation of the Moving Least Squares implict method for surface reconstruction of unordered 3d points. Isosurfaces are extracted from the implict function field using Marching Tetrahedra, a variation of Marching Cubes with significantly less configurations and no ambiguity.

Internally, the reconstruction algorithim is implemented in C++ and uses only the Eigen library as a dependency. CMake is used to build the project and libigl is included only for the purposes of loading the sample bunny.off file and for visualization of the reconstructed surface. No libigl functionality is used in the actual reconstruction. 

To build and run the project, use the following:

```
mkdir build
cd build
make
../recon
```

The algorithm requires vertices and per vertex normals to run. If everything is successful an opengl window should open with the reconstructed mesh.
