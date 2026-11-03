# Volume Renderer

A real-time GPU volume renderer in **C++17 / OpenGL 4.3 / GLSL**. It ray-casts a
scalar volume (synthetic first, real electron-microscopy data later) in a single
fragment-shader pass through a 3D texture, with a transfer function,
gradient-based lighting, an orbit camera, and an interactive parameter UI.

> **Status:** in development. The renderer ray-casts a synthetic volume in a
> single fragment-shader pass — orbit camera, slab-based box traversal,
> front-to-back compositing, and a colour/opacity transfer function with
> step-size-independent opacity are all working. Still to come: gradient-based
> lighting, an interactive parameter UI, and loading a real electron-microscopy
> dataset. Screenshots and dataset provenance will be added as those features land.

## Milestones

The project is built in ordered milestones; each maps to a commit tagged `[M#]`.

- [x] **M0** — Build system, dependencies, and a cleared OpenGL window
- [x] **M1** — Shader pipeline and fullscreen-quad rendering
- [x] **M2** — Orbit camera and per-pixel ray generation
- [x] **M3** — Synthetic volume generation and 3D-texture upload
- [x] **M4** — Single-pass ray-casting (emission–absorption)
- [x] **M5** — Transfer function with opacity correction
- [ ] **M6** — Gradient-based Blinn-Phong lighting
- [ ] **M7** — Interactive parameter UI
- [ ] **M8** — Load and render a real cryo-EM volume
- [ ] **M9** — Marching-cubes isosurface extraction (stretch)

## Requirements

- A C++17 compiler (MSVC 2022 on Windows; GCC/Clang on Linux)
- [CMake](https://cmake.org/) ≥ 3.21
- [Conan](https://conan.io/) 2.x (`pip install "conan>=2.0,<3"`)
- A GPU/driver supporting **OpenGL 4.3 core profile**

## Build

```sh
# 1. One-time: create a Conan profile for your toolchain
conan profile detect --force

# 2. Resolve and build dependencies (C++17 to match the project standard)
conan install . --output-folder=build --build=missing -s compiler.cppstd=17

# 3. Configure and build
cmake --preset conan-default
cmake --build build --config Release

# 4. Run the tests
ctest --test-dir build -C Release --output-on-failure
```

The executable is produced under `build/` (e.g. `build/Release/volume_renderer`
on Windows). Press **Esc** or close the window to exit.

## Dependencies

All third-party libraries are resolved through Conan (`conanfile.txt`):

| Library  | Purpose                          |
|----------|----------------------------------|
| GLFW     | Windowing and input              |
| glad     | OpenGL 4.3 core function loader   |
| GLM      | Vector/matrix math               |
| Dear ImGui | Interactive parameter UI       |
| GoogleTest | Unit tests (via CTest)         |

## License

MIT — see [LICENSE](LICENSE).
