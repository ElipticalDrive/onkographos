# Procedural Terrain Generator with User-Defined Caves

_Made by Namit Bhutani (2022B3A70885H) and Karthik Prakash (2022A7PS0022H)_

This is a OpenGL-based procedural terrain and cave
generator that utilizes noise-based 3D density fields
and the Marching Cubes algorithm for rendering. For
better visuals, basic Phong lighting along with
volumetric fog effects are also implemented.

It uses compute shaders for GPU-accelerated performance
along with a lightweight ImGui-driven viewer that lets
you fly around the scene, tweak noise seeds and edit
procedural caves in real time.

![Main](docs/assets/main.png)

For more details, refer to the docs [here](docs/index.html).

## Technologies & Libraries Used

-   **Language:** C++
-   **Graphics API:** OpenGL
-   **Windowing & Input:** GLFW
-   **OpenGL Function Loading:** GLAD
-   **GUI:** ImGui
-   **Build System:** CMake

## Setup

ImGui and GLAD are included as Git submodules.

```bash
git clone --recursive https://github.com/NamitBhutani/procLan
```

CMake is used to manage the build process.

```bash
mkdir -p build && cd build
cmake ..
cmake --build .
./marchingcubes
```
