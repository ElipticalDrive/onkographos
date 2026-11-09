# Depth-Gradient Shading with Ray-Layer Buffer
Also called: Gradient estimation with ray-layer buffer

## Overview
A technique to estimate surface gradients for shading by analyzing depth coherency in a ray-layer buffer, solving the problem of noisy shading in regions with low volume gradient magnitudes [S1]. It differs from its parent, [Depth-Gradient Shading](6618-depth-gradient-shading.md), by operating on the discrete layers of a multi-layer ray-casting buffer rather than a continuous z-buffer, allowing it to distinguish between different material surfaces encoded in the same pixel column [S1].

## How it works
**Inputs and Outputs:**
- Input: Ray-layer buffer containing depth values and material IDs for each layer; volume gradient field; pixel spacing in world space.
- Output: Blended surface gradient vector per pixel per layer.

**Data Structures:**
The ray-layer buffer is a 3D texture or array where each entry corresponds to a pixel $(x, y)$ and a layer index $l$. Each entry stores:
- `depth`: The depth value of the surface intersection for this layer.
- `materialID`: The identifier of the material at this intersection.

**Steps:**
1. For the current layer $l$ being ray-casted at pixel $(x, y)$, retrieve the entry from the ray-layer buffer. Let this be $E_{curr}$ with depth $d_{curr}$ and material ID $m_{curr}$.
2. Retrieve the entries from the four immediate neighboring pixels $(x-1, y)$, $(x+1, y)$, $(x, y-1)$, and $(x, y+1)$ for the same layer $l$. Let these be $E_{n1}, E_{n2}, E_{n3}, E_{n4}$ with depths $d_{n1}, d_{n2}, d_{n3}, d_{n4}$ and material IDs $m_{n1}, m_{n2}, m_{n3}, m_{n4}$.
3. For each neighbor $i \in \{1, 2, 3, 4\}$, calculate the absolute depth difference: $\Delta d_i = |d_{curr} - d_{ni}|$.
4. Select the neighbor with the minimal absolute depth difference among those that likely belong to the same surface. The source notes that computation is performed on entries encoded with material ID, implying that differences between different material IDs are huge and thus unlikely to be minimal if a same-material neighbor exists. However, the explicit selection criterion given is: select the neighbor $k$ such that $\Delta d_k = \min(\Delta d_1, \Delta d_2, \Delta d_3, \Delta d_4)$. [S1]
5. Calculate the surface gradient magnitude approximation using the selected depth difference and `pixelSpacing` ($p$):
   $$ g_{surf} = \frac{\Delta d_k}{p} $$
   Note: The source describes this as "surface gradient" but the formula yields a scalar magnitude derived from depth difference. In the blending step, it is treated as a scalar component or magnitude. The text says "Calculate the surface gradient using the selected depth difference...". Given the blending formula uses `gradientMagnitude`, it is likely $g_{surf}$ is treated as a scalar magnitude or the norm of a gradient vector approximated by this depth difference. The source does not specify the vector direction, only the magnitude calculation via depth difference. [S1]
6. Retrieve the volume gradient magnitude $g_{vol}$ for the current voxel/sample.
7. Calculate the blend factor:
   $$ blend = \min(1, \frac{g_{vol}}{threshold}) $$
   where `threshold` is a parameter. [S1]
8. Compute the final blended gradient magnitude:
   $$ g_{final} = blend \cdot g_{vol} + (1 - blend) \cdot g_{surf} $$
   The source states "gradient = blend * volumeGradient + (1 - blend) * surfaceGradient". If "gradient" refers to the vector, the direction must be preserved. The source does not specify how the direction of $g_{surf}$ is determined. It only calculates the magnitude from depth difference. It is implied that the volume gradient direction is used, or the surface gradient is only used to modulate the magnitude. Given the context of "shading", the normal direction is crucial. The source says "shading using gradients estimated from the ray-layer buffer". It does not explicitly state that the surface gradient vector replaces the volume gradient vector. It says "blend the surface gradient with the volume gradient". This usually implies vector blending. However, without a defined direction for the surface gradient, only magnitude blending is possible from the given text. The text says "Calculate the surface gradient using the selected depth difference". This produces a scalar. The blending formula uses `gradientMagnitude` to compute `blend`. It is most likely that the volume gradient vector direction is retained, and the magnitude is blended. [S1]

**Parameters:**
- `pixelSpacing`: Distance between neighboring pixels in world space. Assumed isotropic in X and Y. [S1]
- `threshold`: Threshold for blending. Not given in the sources.

**Edge Cases and Pitfalls:**
- If the current pixel or neighbors are empty (no surface), depth values may be invalid. The source mentions that ray segments labeled with `mtIdEmpty` are skipped during ray-layer buffer generation, so empty entries should not exist in the buffer for valid layers. [S1]
- If no neighbor has a similar depth (e.g., at object boundaries), the selected depth difference may be large, leading to inaccurate gradient estimation. The assumption is that "an entry in the ray-layer buffer has at least one direct neighbor that is produced by the same surface". [S1]

## Subtopics
## Operating characteristics
- **Time Complexity:** Constant time per pixel per layer, as it involves a fixed number of texture lookups and arithmetic operations.
- **Memory Complexity:** Requires storage of the ray-layer buffer (depth and material ID per layer per pixel).
- **Strengths:** Reduces noise in shading for regions with low volume gradient magnitudes, such as mask surfaces cutting through volume regions. Provides additional shape hints for masks. [S1]
- **Limitations:** Relies on the assumption of depth coherency among neighbors. May fail at sharp boundaries or isolated surfaces. The direction of the surface gradient is not explicitly defined, limiting its use to magnitude blending or requiring additional assumptions. [S1]
- **When to use:** When shading noisy regions due to low volume gradients, particularly for mask surfaces in multi-layer volume rendering. [S1]
- **When not to use:** When high-fidelity surface normals are required and volume gradients are unreliable, or when the depth coherency assumption is violated.

## Background
Originated in the context of multi-layer volume ray casting on GPU, specifically to address shading artifacts in regions where volume gradients are weak. The method was implemented using OpenGL 2.1 and GLSL on an Nvidia Quadro FX 5600. [S1]

## Sources
[S1] Multi-layer Volume Ray Casting on GPU, https://doi.org/10.2312/vg/vg10/005-012, fetched 2026-10-26
