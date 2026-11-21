# OpenGL Phong Shading

Implements the Phong shading model (ambient + diffuse + specular) on a 3D cube using GLSL vertex and fragment shaders with an interactive camera system.

Built for COSC 4370 (Computer Graphics) at the University of Houston.

## Concepts
- Phong reflection model (L = La + Ld + Ls)
- GLSL vertex and fragment shaders
- Normal matrix transformation for correct lighting
- glm::perspective projection and glm::lookAt camera
- Interactive keyboard/mouse camera navigation

## Files
- `main.cpp` — Cube geometry, projection setup, and render loop
- `Camera.h` — Camera position, orientation, and view matrix
- `phong.vs` — Vertex shader for position and normal transforms
- `phong.frag` — Fragment shader implementing Phong shading
- `Report.pdf` — Implementation writeup with rendered output

## Output
![Phong shaded cube](CUBE.png)
