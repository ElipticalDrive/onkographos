# 2D Texture-Based Volume Rendering with Z-Slice Packing

Also called: 2D Slice Stacking, 2D Texture Z-Slice Packing, SDF Texture Slicing, Volumetric Texture Layout, Z-Slice Packing

## Overview
2D Texture-Based Volume Rendering with Z-Slice Packing is a technique that stores a 3D scalar field as a 2D texture composed of a grid of Z-slices, allowing 3D sampling via 2D texture lookups and trilinear interpolation [S2, unofficial]. It solves the problem of accessing 3D volume data on GPUs that lack efficient 3D texture support or when memory bandwidth for 3D textures is a bottleneck, by mapping the 3D coordinates to 2D UV space [S2, S3, unofficial]. Unlike [Tiled 2D Texture Storage for 3D Fields](2740-tiled-2d-texture-storage-for-3d-fields.md) which tiles the volume into a larger 2D texture, this method specifically arranges Z-slices in a fixed grid layout within a 2D texture to enable standard bilinear filtering for Z-interpolation [S2, unofficial]. It is distinct from [Point Cloud Volumetric Rendering via 2D Atlas](4879-point-cloud-volumetric-rendering-via-2d-atlas.md) as it operates on dense grids rather than point clouds [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: 3D volume data (scalar field), camera position, camera orientation, number of ray marching steps (`steps`), alpha correction factor (`alphaCorrection`).
- Output: 2D color buffer with rendered volume.

**Data Structures:**
1.  **Volume Texture (`cubeTex`):** A 2D texture storing the volume data.
    -   Layout: Z-slices are arranged in a square grid. For a volume with $N$ Z-slices, the texture is sized to accommodate $\lceil\sqrt{N}\rceil \times \lceil\sqrt{N}\rceil$ slices.
    -   Example: For 256 slices, the texture is $256 \times 256$ pixels, with each slice occupying a $16 \times 16$ pixel region (since $16 \times 16 = 256$) [S2, unofficial].
    -   Coordinate Mapping: A 3D coordinate $(x, y, z)$ in a volume of width $W$ is mapped to 2D texture coordinates $(u, v)$ as:
        $$ u = \frac{x + z \cdot W}{\text{TextureWidth}} $$
        $$ v = \frac{y}{\text{TextureHeight}} $$
        Example: For $W=50$, sample $(10, 5, 3)$ maps to $(10 + 3 \cdot 50, 5) = (160, 5)$ in texture space [S3, unofficial].
2.  **Transfer Function Texture (`transferTex`):** A 2D texture mapping opacity (alpha) to RGB color.
3.  **Render Target Texture (`rtTexture`):** A 2D texture storing world-space coordinates of the ray entry point (back face of the volume box).

**Numbered Steps:**

1.  **Preparation (CPU/GPU Upload):**
    -   Pack the 3D volume data into the 2D texture `cubeTex` according to the Z-slice grid layout [S2, unofficial].
    -   Upload the transfer function to `transferTex` [S2, unofficial].
    -   Initialize `rtTexture` with `NearestFilter` to prevent interpolation artifacts at the edges [S2, unofficial].

2.  **First Pass (Back Face Rendering):**
    -   Render the back face of the volume bounding box.
    -   In the vertex shader, compute world-space coordinates of the vertices [S2, unofficial].
    -   In the fragment shader, output the world-space coordinates to `rtTexture` [S2, unofficial].
    -   This pass captures the entry point of the ray into the volume.

3.  **Second Pass (Ray Marching):**
    -   Render the front face of the volume bounding box.
    -   In the vertex shader, compute world-space coordinates and projected coordinates [S2, unofficial].
    -   In the fragment shader, perform ray marching:
        a.  **Ray Initialization:**
            -   Get the entry point (back position) from `rtTexture` using the fragment's texture coordinates [S2, unofficial].
            -   If the back position is invalid (e.g., $(0,0,0)$), discard the fragment [S2, unofficial].
            -   Get the current fragment's world-space position (front position) from the vertex shader [S2, unofficial].
            -   Calculate ray direction: $\vec{dir} = \text{normalize}(\text{backPos} - \text{frontPos})$ [S2, unofficial].
            -   Calculate step size: $\delta = \frac{1.0}{\text{steps}}$ [S2, unofficial].
        b.  **Ray Marching Loop:**
            -   Iterate `steps` times (or up to `MAX_STEPS` to prevent diagonal clipping, e.g., 887 for 512 max steps) [S2, unofficial].
            -   For each step, calculate the current 3D position along the ray.
            -   **Z-Slice Index Calculation:**
                -   Compute the Z-coordinate in texture space $z_{tex}$ (normalized $0.0$ to $1.0$) [S2, unofficial].
                -   Calculate the lower Z-slice index: $z_{slice1} = \lfloor z_{tex} \cdot 255.0 \rfloor$ [S2, unofficial].
                -   Calculate the upper Z-slice index: $z_{slice2} = \min(z_{slice1} + 1.0, 255.0)$ [S2, unofficial].
            -   **Texture Coordinate Adjustment:**
                -   Scale the UV coordinates by the grid size: $u' = u / 16.0, v' = v / 16.0$ [S2, unofficial].
                -   Calculate offsets for $z_{slice1}$:
                    $$ u_1 = u' + \frac{\mod(z_{slice1}, 16.0)}{16.0} $$
                    $$ v_1 = v' + \frac{\floor{(255.0 - z_{slice1}) / 16.0}}{16.0} $$
                -   Calculate offsets for $z_{slice2}$:
                    $$ u_2 = u' + \frac{\mod(z_{slice2}, 16.0)}{16.0} $$
                    $$ v_2 = v' + \frac{\floor{(255.0 - z_{slice2}) / 16.0}}{16.0} $$
            -   **Sampling:**
                -   Sample the volume texture at $(u_1, v_1)$ and $(u_2, v_2)$ using bilinear filtering to get `colorSlice1` and `colorSlice2` [S2, unofficial].
                -   Calculate the fractional Z-distance: $z_{diff} = \mod(z_{tex} \cdot 255.0, 1.0)$ [S2, unofficial].
                -   Interpolate between the slices: $\text{color} = \text{mix}(\text{colorSlice1}, \text{colorSlice2}, z_{diff})$ [S2, unofficial].
            -   **Transfer Function:**
                -   Map the alpha value of the sampled color to the transfer function texture: $\text{rgb} = \text{texture2D}(\text{transferTex}, \text{vec2}(\text{color}.a, 1.0)).\text{rgb}$ [S2, unofficial].
            -   **Alpha Accumulation:**
                -   Calculate the alpha sample: $\alpha_{sample} = \text{color}.a \cdot \text{alphaCorrection} \cdot (1.0 - \alpha_{accumulated}) \cdot \text{alphaScaleFactor}$ [S2, unofficial].
                -   Where $\text{alphaScaleFactor} = 25.6 \cdot \delta$ [S2, unofficial].
                -   Update accumulated alpha and color using front-to-back blending [S2, unofficial].
        c.  **Output:**
            -   Write the final accumulated color to `gl_FragColor` [S2, unofficial].

**Parameters:**
-   `steps`: Number of ray marching steps. Default: 256.0, Range: 0.0-512.0 [S2, unofficial].
-   `alphaCorrection`: Scales alpha to adjust transparency intensity. Default: 1.0, Range: 0.01-5.0 [S2, unofficial].
-   `MAX_STEPS`: Loop limit to prevent clipping. For 512 max steps, set to 887 [S2, unofficial].

**Edge Cases and Pitfalls:**
-   **Edge Interpolation Artifacts:** Interpolated world coordinates from the first pass can produce bogus ray directions at the edges of the cube. Use `NearestFilter` for `rtTexture` to mitigate this [S2, unofficial].
-   **Invalid Back Position:** If the back position is $(0,0,0)$, the fragment is discarded [S2, unofficial].
-   **Diagonal Clipping:** The ray length through the volume can exceed 1.0 in normalized space. Use `MAX_STEPS` to ensure the ray reaches the back face [S2, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(\text{steps} \cdot \text{resolution})$ per fragment, where `steps` is the number of ray marching iterations.
-   **Memory Complexity:** $O(\text{volume\_size})$ for the 2D texture storage. The 2D texture size is $\lceil\sqrt{N}\rceil \times \lceil\sqrt{N}\rceil \times \text{slice\_resolution}^2$ for $N$ slices.
-   **Performance:** Efficient for GPUs with strong 2D texture filtering capabilities. The use of bilinear filtering for Z-interpolation reduces the need for manual interpolation in the shader [S2, unofficial].
-   **Strengths:** Simple implementation, leverages standard 2D texture hardware, good for dense volumes [S2, unofficial].
-   **Limitations:** Memory usage can be high for large volumes due to the 2D texture size. Not suitable for highly sparse volumes without additional packing strategies [S1, unofficial].
-   **When to Use:** When 3D texture support is limited or when the volume is dense and fits within the 2D texture size constraints [S2, unofficial].
-   **When Not to Use:** For highly sparse volumes where packing efficiency is critical [S1, unofficial].

## Background
-   **Origin:** The technique is described in the context of web-based volume rendering using WebGL [S2, unofficial].
-   **Development:** It builds on the concept of storing 3D data in 2D textures, as discussed in [Tiled 2D Texture Storage for 3D Fields](2740-tiled-2d-texture-storage-for-3d-fields.md) [S2, unofficial].
-   **Usage:** Used in web-based volume rendering applications where 3D texture support may be inconsistent or inefficient [S2, unofficial].

## Sources
[S1] A Hybrid CPU-GPU Approach for Memory-Efficient Volume Rendering of Medical Image Data, https://www.researchgate.net/publication/408581180_A_Hybrid_CPU-GPU_Approach_for_Memory-Efficient_Volume_Rendering_of_Medical_Image_Data, fetched 2026-10-26 (unofficial)
[S2] Maayan-Moshe/web_volume_rendering, https://github.com/Maayan-Moshe/web_volume_rendering, fetched 2026-10-27 (unofficial)
[S3] Signed Distance Field Rendering Journey pt.1 – kosmonaut's blog, https://kosmonautblog.wordpress.com/2017/05/01/signed-distance-field-rendering-journey-pt-1/, fetched 2026-10-29 (unofficial)
