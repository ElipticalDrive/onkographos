# GPU Implementation Architecture

Also called: OpenGL/GLSL Volume Rendering Implementation

## Overview
This article describes the standard software architecture for implementing direct volume ray casting on modern GPUs using OpenGL and GLSL. It details the pipeline for generating spherical harmonic (SH) coefficient volumes and performing ray marching via fragment shaders. It distinguishes itself from generic hardware-accelerated approaches by focusing on the specific data structures (3D coefficient volumes) and rendering techniques (slice-by-slice ray marching with Monte Carlo sampling) required for advanced light-material interaction [S1].

## How it works
The implementation uses C++ with OpenGL and GLSL [S1]. The core challenge addressed is the integration of three functions (material function $M$, incoming light $L_{mi}$, and outgoing light $L_{mi}^0$) which is complex for direct SH integration. The solution approximates this integration to achieve convincing visual results [S1].

**Inputs and Outputs:**
*   **Input:** Volume data, material properties, lighting conditions (represented as SH coefficients), and pre-generated random sample directions.
*   **Output:** Rendered image with advanced lighting interactions, specifically SH coefficient volumes for $L_{mi}$ and $L_{mi}^0$ [S1].

**Data Structures:**
1.  **3D Coefficient Volumes:** SH coefficients for $L_{mi}$ and $L_{mi}^0$ are stored in 3D texture volumes [S1].
2.  **2D Textures:** Used for uploading pre-computed data from the CPU, including:
    *   Normalized Cartesian vectors for Monte Carlo sampling.
    *   Spherical Harmonic basis functions $Y_{lm}(\vartheta, \phi)$ for each sample direction [S1].

**Algorithm Steps:**
1.  **Generate SH Coefficient Volumes:**
    *   Use Framebuffer Objects (FBOs) in conjunction with Multiple Render Targets (MRT) to generate the 3D coefficient volumes [S1].
    *   Note: Coefficients for $L_{mi}$ can be discarded after $L_{mi}^0$ has been computed, as they are not accessed during rendering [S1].
2.  **Prepare Sampling Data (CPU Side):**
    *   Generate random samples for Monte Carlo sampling on the CPU as normalized Cartesian vectors [S1].
    *   Compute $Y_{lm}(\vartheta, \phi)$ for each sample direction [S1].
    *   Upload these vectors and basis functions to the GPU as 2D textures [S1].
3.  **Ray Marching (Fragment Shader):**
    *   Render the volume slice by slice [S1].
    *   In the fragment shader, use the uploaded random samples to cast rays [S1].
    *   Perform the approximate integration of light-material interaction [S1].
    *   After a ray terminates, multiply its contribution with the values $Y_{lm}(\vartheta_s, \phi_s)$ obtained from the second 2D texture [S1].

**Parameters:**
*   **Samples:** The number of samples affects quality. Example: 1250 samples distributed over the hemisphere [S1].
*   **Implementation Language:** C++ with OpenGL and GLSL [S1].

**Edge Cases and Pitfalls:**
*   **Random Number Generation:** The GLSL specification describes functionality for generating random numbers, but this feature was not implemented in the referenced environment. Therefore, random samples must be generated on the CPU and uploaded [S1].
*   **Approximation:** The integration technique used is not physically correct but provides convincing visual results [S1].

## Subtopics

### GPU Direct Volume Rendering with Out-of-Core Paging
Direct volume rendering on GPUs that integrates rays by accumulating gradients across sequential cutting planes of the volume stored as 2D textures, bypassing GPU memory limits by paging data from CPU main memory [S2, unofficial].
*   **Definition:** Stores the volume out-of-core in CPU main memory and pages it into GPU memory [S2, unofficial].
*   **Performance:** Medium-sized volumes up to $512^3$ can be viewed at interactive rates [S2, unofficial].
*   **Context:** This approach addresses the limitation that larger volumes are restricted by GPU memory, unlike standard in-core rendering [S2, unofficial].

## Operating characteristics
*   **Memory:** Standard implementations are limited by GPU memory capacity. Out-of-core variants mitigate this by paging data [S2, unofficial].
*   **Performance:** GPU volume rendering is fast, permitting interactive rates for medium-sized volumes ($512^3$) [S2, unofficial].
*   **Complexity:** Handling three-function SH integration is complex; the described architecture uses an approximation to maintain performance [S1].

## Background
The concepts described in this paper have been integrated into a GPU-based volume ray-caster implemented in C++ using OpenGL and GLSL [S1]. The approach builds on the effectiveness of direct volume rendering on GPUs, which is faster than ray tracing for this purpose [S2, unofficial].

## Sources
[S1] Advanced Light Material Interaction for Direct Volume Rendering, https://doi.org/10.2312/vg/vg10/101-108, fetched 2026-10-26
[S2] PDF A Survey of Octree Volume Rendering Methods, https://www.sci.utah.edu/~knolla/octsurvey.pdf, fetched 2026-10-29 (unofficial)
