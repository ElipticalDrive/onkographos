# 1D Volume Ray Casting with Z-Buffer Clipping

Also called: CUDA 1D Volume Ray Caster, vtkCUDAVolumeMapper

## Overview
A GPU-accelerated volume rendering technique that casts rays through a 3D volume using a pre-computed Z-buffer to determine ray entry and exit points, and 1D transfer functions for opacity and color mapping [S1, unofficial]. It differs from its sibling [GPU Volume Rendering with Shared Memory Caching and Phong Shading](21421-gpu-volume-rendering-with-shared-memory-caching-and-phong-sh.md) by relying on hardware Z-buffer clipping for early ray termination and boundary culling rather than shared memory caching for texture access optimization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volume texture (`CUDA_vtkCUDA1DVolumeMapper_input_texture`), 1D transfer function textures (alpha, color R/G/B), Z-buffer texture (`zbuffer_texture`), clipping planes, view-to-voxels matrix, volume bounds, and spacing.
*   **Output:** Composited RGBA color and opacity for each pixel.

**Data Structures:**
*   `cudaVolumeInformation` (`volInfo`): Contains volume bounds (`Bounds[0..5]`), spacing (`Spacing[0..2]`), and minimum spacing (`MinSpacing`).
*   `cudaRendererInformation` (`renInfo`): Contains number of clipping planes (`NumberOfClippingPlanes`), array of clipping planes (`ClippingPlanes`, array of `float4`), and the view-to-voxels transformation matrix (`ViewToVoxelsMatrix`, 16 floats) [S1, unofficial].
*   `cudaOutputImageInformation` (`outInfo`): Contains image resolution (`resolution`, `int2`) and per-ray intermediate data arrays: `rayStartX/Y/Z`, `rayIncX/Y/Z`, and `numSteps` [S1, unofficial].
*   **Textures:**
    *   `zbuffer_texture`: `cudaArray*` bound to `texture<float, 2, cudaReadModeElementType>`. Parameters: normalized coordinates, point filtering, clamp addressing [S1, unofficial].
    *   Transfer functions: `alpha_texture_1D`, `galpha_texture_1D`, `colorR_texture_1D`, `colorG_texture_1D`, `colorB_texture_1D` (read-only 1D textures) [S1, unofficial].
    *   Volume: `CUDA_vtkCUDA1DVolumeMapper_input_texture` (3D texture) [S1, unofficial].

**Parameters:**
*   `BLOCK_DIM2D`: Block dimension for 2D grid. Default 16. Range [4, 16] [S1, unofficial].
*   `dRandomRayOffsets`: Array of size `BLOCK_DIM2D * BLOCK_DIM2D` floats, used for de-artefacting [S1, unofficial].

**Algorithm Steps:**

1.  **Ray Setup:** For each pixel $(x, y)$, compute normalized view coordinates:
    $$ \text{viewRayX} = 1.0 - \frac{x}{\text{outInfo.resolution.x}} $$
    $$ \text{viewRayY} = \frac{y}{\text{outInfo.resolution.y}} $$
    Sample the Z-buffer to get the depth of the front surface:
    $$ \text{endDepth} = \text{tex2D}(zbuffer\_texture, 1.0 - \text{viewRayX}, \text{viewRayY}) $$
    [S1, unofficial]

2.  **Ray Origin/End Calculation:** Transform normalized coordinates and depth into voxel space using the `ViewToVoxelsMatrix` $M$ (16 floats, column-major assumed based on formula structure):
    $$ \text{startNorm} = \text{viewRayX} \cdot M[12] + \text{viewRayY} \cdot M[13] + M[15] $$
    $$ \text{rayStart}.x = \text{viewRayX} \cdot M[0] + \text{viewRayY} \cdot M[1] + M[3] $$
    $$ \text{rayStart}.y = \text{viewRayX} \cdot M[4] + \text{viewRayY} \cdot M[5] + M[7] $$
    $$ \text{rayStart}.z = \text{viewRayX} \cdot M[8] + \text{viewRayY} \cdot M[9] + M[11] $$
    $$ \text{rayEnd}.x = \text{rayStart}.x + \text{endDepth} \cdot M[2] $$
    $$ \text{rayEnd}.y = \text{rayStart}.y + \text{endDepth} \cdot M[6] $$
    $$ \text{rayEnd}.z = \text{rayStart}.z + \text{endDepth} \cdot M[10] $$
    $$ \text{endNorm} = \text{startNorm} + \text{endDepth} \cdot M[14] $$
    [S1, unofficial]

3.  **Clipping against Clipping Planes:** For each plane $i$ from $0$ to `NumberOfClippingPlanes`-1:
    Let plane $P_i$ be defined by normal $\mathbf{n}_i$ and distance $d_i$ (stored in `ClippingPlanes[i]` as `float4`).
    $$ \mathbf{rayDir} = \text{rayEnd} - \text{rayStart} $$
    $$ dp = \mathbf{n}_i \cdot \mathbf{rayDir} $$
    $$ t = -\frac{\mathbf{n}_i \cdot \text{rayStart} + d_i}{dp} $$
    If $0 < t < 1$:
    *   If $dp > 0$, the ray is entering the half-space defined by the plane. Update `rayStart` to $\text{rayStart} + t \cdot \mathbf{rayDir}$.
    *   If $dp < 0$, the ray is exiting the half-space. Update `rayEnd` to $\text{rayStart} + t \cdot \mathbf{rayDir}$.
    *   If $dp = 0$, the ray is parallel to the plane. The division results in infinity/NaN. The condition $0 < t < 1$ typically fails, skipping the update [S1, unofficial].
    [S1, unofficial]

4.  **Clipping against Volume Bounds:** Define bounds with a 1.0f offset to account for texture sampling boundaries:
    $$ \text{bounds}_0 = \text{volInfo.Bounds}[0] + 1.0, \quad \text{bounds}_1 = \text{volInfo.Bounds}[1] - 1.0 $$
    $$ \text{bounds}_2 = \text{volInfo.Bounds}[2] + 1.0, \quad \text{bounds}_3 = \text{volInfo.Bounds}[3] - 1.0 $$
    $$ \text{bounds}_4 = \text{volInfo.Bounds}[4] + 1.0, \quad \text{bounds}_5 = \text{volInfo.Bounds}[5] - 1.0 $$
    For each axis $k \in \{x, y, z\}$:
    *   If $\text{rayDir}.k > 0$:
        $$ \text{diffS} = \frac{\text{bounds}_{2k} - \text{rayStart}.k}{\text{rayDir}.k} $$
        $$ \text{diffE} = \frac{\text{bounds}_{2k+1} - \text{rayStart}.k}{\text{rayDir}.k} $$
    *   Else ($\text{rayDir}.k \le 0$):
        $$ \text{diffS} = \frac{\text{bounds}_{2k+1} - \text{rayStart}.k}{\text{rayDir}.k} $$
        $$ \text{diffE} = \frac{\text{bounds}_{2k} - \text{rayStart}.k}{\text{rayDir}.k} $$
    Update ray start and end:
    $$ \text{rayStart}.k \leftarrow \text{rayStart}.k + \text{rayDir}.k \cdot \max(0.0, \text{diffS}) $$
    $$ \text{rayEnd}.k \leftarrow \text{rayStart}.k + \text{rayDir}.k \cdot \max(0.0, \text{diffE}) $$
    If the ray is still outside the bounds after clipping (i.e., `rayStart` > `rayEnd` in any dimension or effectively zero length), set `rayStart = rayEnd` to skip sampling [S1, unofficial].

5.  **Step Calculation:** Compute the number of steps based on the minimum voxel spacing to ensure uniform sampling:
    $$ \text{lenSq} = (\text{rayInc}.x)^2 \cdot \text{volInfo.Spacing}.x^2 + (\text{rayInc}.y)^2 \cdot \text{volInfo.Spacing}.y^2 + (\text{rayInc}.z)^2 \cdot \text{volInfo.Spacing}.z^2 $$
    *(Note: `rayInc` here refers to the vector `rayEnd - rayStart` before normalization)*
    $$ \text{numSteps} = \frac{\sqrt{\text{lenSq}}}{\text{volInfo.MinSpacing}} $$
    Normalize the direction vector for stepping:
    $$ \text{rayInc}.x \leftarrow \frac{\text{rayInc}.x}{\text{numSteps}}, \quad \text{rayInc}.y \leftarrow \frac{\text{rayInc}.y}{\text{numSteps}}, \quad \text{rayInc}.z \leftarrow \frac{\text{rayInc}.z}{\text{numSteps}} $$
    [S1, unofficial]

6.  **Ray Casting Loop (1D):**
    Initialize output color $C_{out} = (0,0,0,0)$ and output opacity $\alpha_{out} = 0$.
    Current position $\mathbf{p} = \text{rayStart}$.
    Loop $i$ from $0$ to $\text{numSteps}$:
    *   Sample volume at $\mathbf{p}$: $v = \text{tex3D}(CUDA\_vtkCUDA1DVolumeMapper\_input\_texture, \mathbf{p})$.
    *   Map $v$ to source color $C_{src}$ and source opacity $\alpha_{src}$ using 1D transfer function textures [S1, unofficial].
    *   Front-to-back compositing:
        $$ C_{out} = C_{in} + C_{src} \cdot \alpha_{src} \cdot (1 - \alpha_{out}) $$
        $$ \alpha_{out} = \alpha_{out} + \alpha_{src} \cdot (1 - \alpha_{out}) $$
    *   If $\alpha_{out} \ge 1.0$, break loop.
    *   $\mathbf{p} \leftarrow \mathbf{p} + \text{rayInc}$
    [S1, unofficial]

**Pitfalls:**
*   **Division by Zero:** In clipping, if $dp = 0$, division by zero occurs. The implementation relies on the subsequent $t$ range check to ignore the invalid intersection, but this may produce NaNs/Infs if not handled by the hardware/floating-point status flags [S1, unofficial].
*   **Zero-Length Rays:** If clipping results in `rayStart` being outside `rayEnd`, the ray is set to zero length. This prevents sampling but requires careful handling of the loop condition [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the average number of steps per ray (determined by volume thickness and minimum spacing).
*   **Memory Complexity:** $O(V)$ for the volume texture, $O(T)$ for transfer functions, and $O(P)$ for clipping planes. Intermediate storage for ray parameters is $O(N)$ per ray attribute.
*   **Performance:** Performance is heavily dependent on the Z-buffer hit rate. Early termination via Z-buffer clipping significantly reduces the average number of steps compared to full-volume traversal.
*   **Strengths:** Simple implementation, leverages existing OpenGL Z-buffer for accurate front-surface hit detection, efficient culling of rays that miss the volume or are occluded.
*   **Limitations:** Requires a pre-computed Z-buffer (from a previous pass or geometry rendering). Does not use shared memory caching, potentially leading to higher memory bandwidth usage for texture lookups compared to sibling methods.
*   **When to use:** When accurate front-surface clipping is required and a Z-buffer is available. Suitable for static volumes or when the overhead of shared memory management is not justified.
*   **When not to use:** When shared memory caching is critical for performance on large volumes, or when Z-buffer information is not available.

## Background
This method is part of the CUDA Volume Rendering module in 3D Slicer, specifically implemented in `vtkCUDAVolumeMapper` [S1, unofficial]. It was developed by John Stuart Haberl Baxter (Dr. Peter's Lab at Robarts Research Institute) and first documented on March 28, 2011 [S1, unofficial]. It is used in medical imaging visualization within the Slicer platform.

## Sources
[S1] CUDAVolumeRendering, https://github.com/finetjul/CUDAVolumeRendering, fetched 2026-10-27 (unofficial)
