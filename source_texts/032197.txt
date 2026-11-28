# Canyon: WebGL Voxel Engine

A custom-built WebGL 2 rendering engine that procedurally generates massive 3D voxel landscapes using hardware instancing and image sampling. Built entirely from scratch without external frameworks like Three.js.

## Overview
This project was designed to explore low-level computer graphics principles by directly interacting with the GPU. It renders a 500x500 grid (250,000 cubes) at a high framerate by utilizing WebGL 2 hardware instancing, packing all translation, scaling, and color data into flat `Float32Arrays` for a single, hyper-optimized draw call.

## Features
*   **Custom Math Library:** Implements custom matrix operations (translation, scaling, perspective projection, and lookAt) to handle 3D transformations natively in JavaScript.
*   **Hardware Instancing (WebGL 2):** Bypasses CPU bottlenecks by sending a single 2x2x2 cube primitive to the GPU and executing `gl.drawElementsInstanced` to render hundreds of thousands of copies simultaneously.
*   **Procedural Image Sampling:** Extracts raw pixel arrays from 2D `.png` files (a grayscale heightmap and an RGB colormap) to mathematically dictate the exact Y-axis height and vertex colors of each voxel.
*   **Atmospheric Shaders:** Custom GLSL fragment shader utilizing a `smoothstep` function based on vertex depth (`vPosition`) to render distance fog and blend distant geometry with the background sky.
*   **Parametric Animation:** Evaluates cubic Bezier curves in the render loop to drive the smooth, looping flight paths of an ambient skyscape ("bird" particles).

## Installation & Usage
Because this engine relies on JavaScript's `getImageData()` to sample local image files, it must be run through a local web server to avoid browser CORS (Cross-Origin Resource Sharing) restrictions.

1. Clone the repository
2. Open the project folder in VSCode.
3. Install the **Live Server** extension.
4. Right-click `index.html` and select **Open with Live Server**.

## File Structure
*   `index.html`: The application shell and WebGL `<canvas>` container.
*   `style.css`: Viewport and UI styling.
*   `script.js`: The core engine, handling WebGL context, GLSL shader compilation, matrix math, buffer allocation, image sampling, and the requestAnimationFrame loop.
*   `assets/`: Contains the `heightmap.png` and `colormap.png` used for procedural terrain generation.

## Technical Details
*   **Language:** JavaScript (ES6+), GLSL (OpenGL Shading Language)
*   **Graphics API:** WebGL 2
*   **Dependencies:** None (Zero external libraries)

## Demo Photo
<img width="911" height="611" alt="Screenshot 2026-10-21 191523" src="https://github.com/user-attachments/assets/f8d5c522-d77f-4fac-9909-e72cde38e0b4" />
