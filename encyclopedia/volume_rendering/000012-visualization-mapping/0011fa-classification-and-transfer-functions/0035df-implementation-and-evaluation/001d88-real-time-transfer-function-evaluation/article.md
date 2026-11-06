# Real-time transfer function evaluation

Also called: Engel, Kraus & Ertl Technique, Texture-based volume rendering with GPU illumination

## Overview
This method evaluates and modifies the transfer function in real-time by storing pre-evaluated classification results (color and opacity) in 2D textures, allowing for dynamic GPU-based illumination. It distinguishes itself from CPU-based illumination techniques (such as Westermann & Sevenich) by performing lighting calculations dynamically on the GPU, enabling interactive modification of lighting parameters during rendering [S1, unofficial].

## How it works
**Inputs:**
*   Volume data grid.
*   Transfer function parameters (uniforms modifiable from the application).
*   Illumination parameters (light positions, material properties).

**Outputs:**
*   Rendered image with primary color and opacity for constituent voxels.

**Data Structures:**
1.  **2D Texture (Transfer Function Lookup):** Stores the result of the transfer function evaluated for a range of input values. The texture maps input scalar values to RGBA output values (primary color and opacity) [S1, unofficial].
2.  **Compressed RGB Texture (Gradient Storage):** Stores the pre-calculated gradient for each volume element. The gradient is compressed into an RGB texture format [S1, unofficial].

**Steps:**
1.  **Pre-compute Gradients:** Calculate the gradient for each volume element beforehand. Store these gradients in a compressed RGB texture format [S1, unofficial].
2.  **Build Transfer Function Texture:** Evaluate the transfer function for a range of input values. Store the resulting color and opacity pairs in a 2D texture [S1, unofficial].
3.  **Ray Casting with Classification:** Perform ray casting. For each sample point, use the 2D texture to classify the volume data, retrieving the primary color and opacity for the constituent voxels [S1, unofficial].
4.  **GPU Illumination:** Use a fragment program (e.g., Cg) to calculate the final color value for each volume element. This calculation takes into account diffuse and specular contributions based on the pre-computed gradients and uniform parameters modifiable interactively from the C application [S1, unofficial].

**Parameters:**
*   **Gradient Storage:** Compressed into an RGB texture [S1, unofficial].
*   **Illumination:** Dynamic, performed on the GPU using Cg [S1, unofficial].

**Pitfalls:**
*   The original work's results could not be reproduced because the plane reduction optimization was not implemented due to the complexity of their approximation towards two-texture interpolation [S1, unofficial].

## Subtopics
## Operating characteristics
*   **Performance:** Achieves 4 to 6 frames per second with GPU-based illumination and shading [S1, unofficial].
*   **Strengths:** Enables dynamic lighting on the volume, which is an advantage over techniques that calculate illumination once on the CPU [S1, unofficial].
*   **Limitations:** The original plane reduction optimization was not implemented, potentially impacting performance or complexity [S1, unofficial].

## Background
The technique was proposed by Engel, Kraus, and Ertl. It is discussed in the context of texture mapping based volume rendering [S1, unofficial].

## Sources
[S1] Texture Mapping Based Volume Rendering, https://www.ciroduran.com/files/Tesis.pdf, fetched 2026-10-29 (unofficial)
