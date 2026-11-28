# Ray Tracer in C++

This project implements a basic ray tracer in C++ that supports the rendering of scenes with spheres, triangles, and meshes. The ray tracer computes intersections of camera rays with geometric primitives, handles shadows, and applies lighting to generate the final image from the given vectors. Below are the input1 and input2 results:

<img width="632" height="627" alt="image" src="https://github.com/user-attachments/assets/57110634-8c94-4d8b-a635-2941ebe88cd1" />

And

<img width="579" height="538" alt="image" src="https://github.com/user-attachments/assets/471b8603-98b0-4610-ae74-95c91f2c908e" />

## Features

- **Mesh Parsing**: The code can parse mesh data from a file and store it in a vector of tuples.
- **Sphere Intersection**: Computes intersections between rays and spheres, taking into account the sphere's material and center.
- **Triangle Intersection**: Computes intersections between rays and triangles, using the Möller-Trumbore intersection algorithm.
- **Mesh Intersection**: Handles intersections between rays and arbitrary meshes composed of triangles.
- **Shadow Computation**: Determines whether a point is in shadow by tracing shadow rays from the intersection point to the light sources.
- **Lighting Calculation**: Computes the final color of a point using ambient, diffuse, and specular lighting models.

## File Structure

- **`parseMeshes`**: Reads mesh data from a file and stores it in a vector of tuples.
- **`computeSphereIntersection`**: Calculates the intersection of a ray with a sphere.
- **`intersectRayTriangle`**: Determines if a ray intersects a triangle using the Möller-Trumbore algorithm.
- **`computeTriangleIntersection`**: Computes the intersection of a ray with a triangle and retrieves the corresponding material.
- **`computeMeshIntersection`**: Calculates the intersection of a ray with a mesh.
- **`computeCameraRay`**: Casts a ray from the camera and determines the closest intersection with any object in the scene.
- **`computeShadow`**: Checks if an intersection point is in shadow by tracing a shadow ray towards the light source.
- **`computeLighting`**: Computes the lighting at the intersection point using the Phong reflection model.

## Dependencies

- **C++ Standard Library**: The code uses standard C++ libraries such as `<vector>`, `<tuple>`, `<cmath>`, and `<limits>`.
- **stb_image_write** ([`stb_image_write.h`](stb_image_write.h)): public-domain single header included from [`Raytracer.cpp`](Raytracer.cpp) to write **`output.png`** (no extra link flags).

## How to Run

1. **Compile** (single translation unit). Use **`-static`** so the `.exe` runs from PowerShell / Explorer without MSYS2 on `PATH` (otherwise MinGW DLLs are missing and the process exits with `0xC0000139`):
   ```sh
   g++ -std=c++17 -O2 -Wall -Wextra -static -o Raytracer.exe Raytracer.cpp
   ```
   Or use `make` (same flags). If **clangd** in Cursor still reports missing headers, set the C++ compiler to your MinGW `g++` or adjust [`.clangd`](.clangd) paths when you upgrade GCC.

2. **Execute** with a scene file (writes **`output.png`** in the current directory). From **MSYS2 UCRT64**, you can instead add `C:\msys64\ucrt64\bin` to `PATH` and link without `-static` if you prefer a smaller binary.
   ```sh
   ./Raytracer input1.txt
   ```

3. The program reads the scene file, computes intersections, and generates **`output.png`** from the camera, lights, and materials.

## Input Files

- **Mesh File**: The code expects a mesh file that contains mesh definitions. Each mesh is defined by a series of vertex indices and material properties.

## Example Usage

An example of how to use the code to parse a mesh file and compute intersections is provided in the main function. Customize the scene by modifying the mesh file, camera settings, and lighting configuration.
