# Quaternion Julia Set Raymarcher

A real-time GPU renderer for exploring 3D slices of quaternion Julia sets. The project evaluates a four-dimensional fractal distance estimator in an OpenGL fragment shader, then uses sphere tracing, normal estimation, and Phong lighting to turn the result into an interactive 3D surface.

![A green quaternion Julia set rendered against a gray background, with live shader controls](https://github.com/user-attachments/assets/0a72c3d5-b154-4215-9161-cd666a224f99)

## Highlights

- Renders the fractal entirely on the GPU with a GLSL 4.30 fragment shader
- Iterates `q = q^2 + c` in quaternion space and visualizes a 3D slice
- Uses a distance estimator and sphere tracing for efficient surface intersections
- Skips empty rays with an analytical bounding-sphere intersection
- Reconstructs surface normals with finite differences and applies Phong lighting
- Supports stochastic supersampling for smoother edges
- Exposes the Julia constant, field of view, hit epsilon, sample count, and iteration limit through Dear ImGui
- Rotates the fractal interactively without rebuilding geometry

## How It Works

The application draws a single full-screen quad. For every pixel, the fragment shader constructs a camera ray and tests it against a sphere enclosing the fractal. Rays that enter the sphere are advanced using a quaternion Julia-set distance estimate until they reach the surface or leave the bounded volume.

At a hit, the shader samples the distance field around the intersection to estimate a normal. A compact Phong lighting pass shades the surface, and optional sub-pixel jitter averages multiple samples to reduce aliasing.

```text
full-screen quad
      |
camera ray per fragment
      |
bounding-sphere test
      |
quaternion distance estimation
      |
sphere-traced surface hit
      |
normal estimation + lighting
      |
anti-aliased pixel
```

## Controls

| Input | Action |
| --- | --- |
| `W` / `S` | Rotate the fractal up / down |
| `A` / `D` | Rotate the fractal left / right |
| `Esc` | Close the application |
| ImGui panel | Tune fractal and render parameters live |

The render controls trade fidelity for speed: a smaller epsilon produces finer surface detail, while higher iteration and anti-aliasing sample counts improve quality at a greater GPU cost.

## Tech Stack

- C++
- OpenGL 4.3 and GLSL 4.30
- GLFW for the window and OpenGL context
- GLAD for OpenGL function loading
- GLM for camera and matrix math
- Dear ImGui for real-time controls

## Running Locally

This repository currently contains the renderer source and shaders, but does not yet include a generated project or cross-platform build configuration. To integrate it into a local OpenGL project, provide GLFW, GLAD, GLM, and Dear ImGui (including its GLFW and OpenGL 3 backends), then compile the files in `src/` and link the required platform OpenGL libraries.

The renderer requests an OpenGL 4.3 core context. Run the resulting executable from the repository root so the relative shader paths resolve:

```text
shaders/render.vert
shaders/juliaSet.frag
```

## Project Structure

```text
.
|-- shaders/
|   |-- juliaSet.frag   # Raymarching, quaternion math, normals, and lighting
|   `-- render.vert     # Full-screen quad vertex shader
`-- src/
    |-- main.cpp        # Window, render loop, controls, and quad setup
    |-- camera.*        # Camera state and rotation matrices
    |-- shader.*        # Shader compilation, linking, and uniforms
    |-- common.h        # Shared graphics and math includes
    `-- glad.c          # OpenGL loader implementation
```

## License

Released under the [MIT License](LICENSE).
