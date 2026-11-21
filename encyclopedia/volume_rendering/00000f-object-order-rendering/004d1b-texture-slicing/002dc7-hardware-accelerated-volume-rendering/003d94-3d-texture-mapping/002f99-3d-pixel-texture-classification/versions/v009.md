# 3D Pixel Texture Classification

Also called: VGH Lookup

## Overview
3D Pixel Texture Classification is a hardware-accelerated volume rendering technique that maps voxel properties to optical attributes using a three-dimensional lookup texture. It solves the problem of efficient, real-time classification by offloading the transfer function evaluation to the GPU's texture hardware, allowing color and opacity to be determined in a single texture fetch per sample [S1]. It differs from standard 1D or 2D transfer functions by incorporating gradient magnitude and second derivative information directly into the sampling coordinate, enabling sophisticated boundary-aware classification without complex shader arithmetic [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset where each voxel contains at least three values: Data Value ($D$), Gradient Magnitude ($G$), and Second Derivative ($S$).
*   **Output:** A 2D framebuffer containing the final rendered image with color $C$ and accumulated opacity $\alpha$.

**Data Structures:**
The core data structure is a 3D pixel texture (lookup table) with dimensions $N_D \times N_G \times N_S$.
*   **Axis 1 (Data Value):** 8-bit resolution (256 entries).
*   **Axis 2 (Gradient Magnitude):** 8-bit resolution (256 entries).
*   **Axis 3 (Second Derivative):** Limited resolution (e.g., 3 entries), representing "sheets" for negative, zero, and positive values.
*   **Texture Format:** Each texel stores 32-bit floating-point color ($R, G, B$) and 8-bit or 32-bit opacity ($\alpha$).

**Preprocessing:**
1.  **Gradient Calculation:** Compute the gradient magnitude $G$ for each voxel.
2.  **Second Derivative Calculation:** Compute the second derivative $S$.
3.  **Sign Maintenance:** Since $S$ is signed, apply a scale and bias to map negative, zero, and positive values to distinct, interpolatable regions in the texture coordinate space [S1].

**Numbered Steps:**
1.  **Ray Generation:** Generate rays from the camera through the volume bounding box using the shared principles of [Hardware-Accelerated Volume Rendering](5616-hardware-accelerated-volume-rendering.md).
2.  **Ray Marching:** For each sample point along the ray:
    a. Sample the volume data to retrieve $D$, $G$, and $S$.
    b. Normalize these values to texture coordinates $(u, v, w)$ within $[0, 1]$.
    c. Perform a 3D texture lookup using $(u, v, w)$ to retrieve pre-classified color $C_i$ and opacity $\alpha_i$.
3.  **Compositing:** Accumulate the result using front-to-back compositing:
    $$
    C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i
    $$
    $$
    \alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
    $$
4.  **Early Ray Termination:** Terminate the ray if $\alpha' > 0.95$ [S1, S2].

**Parameters:**
*   **Resolution:** 256x256 for Data Value and Gradient Magnitude axes.
*   **Second Derivative Control:** Three control points (negative, zero, positive) [S1].
*   **Boundary Emphasis:** A slider that scales the opacity of the outer sheets (non-zero second derivatives) relative to the center sheet (zero second derivative) [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Full 8-bit resolution on all three axes would be prohibitively large for hardware textures; thus, the second derivative axis resolution is limited [S1].
*   **Interpolation:** The second derivative axis must maintain sign information via scale and bias to ensure correct interpolation across the zero-crossing boundary [S1].

## Subtopics

### Adaptive Pre-integrated Voxel Classification
A technique included in a real-time raycaster built by Roettger et al. to exploit the intrinsically parallel nature of GPUs, along with volume clipping. It addresses the inefficiency of multi-pass rendering pipelines by integrating adaptive classification into a single-pass framework [S2].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of samples per ray. Classification is $O(1)$ per sample due to hardware texture lookup.
*   **Memory Complexity:** $O(N_D \times N_G \times N_S)$ for the 3D lookup texture. With 256x256x3 dimensions, this is manageable.
*   **Performance:** Enables real-time rendering on modern GPUs by minimizing shader arithmetic [S2].
*   **Strengths:** High performance, supports complex multi-dimensional transfer functions, leverages hardware interpolation.
*   **Limitations:** Texture size constraints limit the resolution of higher-order derivative axes; pre-computation of gradients and second derivatives is required [S1].

## Background
Originated in the early 2000s with the work of Kindlmann et al. (Simian), this method was designed to leverage the emerging capabilities of GPU texture units for volume visualization [S1]. It was later extended in real-time raycasting frameworks by Roettger et al. and Stegmaier et al. to improve efficiency and support spectral DVR [S2].

## Sources
[S1] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
