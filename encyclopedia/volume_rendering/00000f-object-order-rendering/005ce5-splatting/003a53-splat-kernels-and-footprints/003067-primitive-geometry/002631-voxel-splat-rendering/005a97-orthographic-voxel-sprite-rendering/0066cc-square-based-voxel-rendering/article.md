# Square-based voxel rendering

Also called: 2D square mapping, Simplified voxel rendering

## Overview
Square-based voxel rendering is a technique for small voxel sets that maps 3D voxels to 2D squares (tiles) on the screen to achieve real-time performance with visual compromises, avoiding the GPU load of dense polygon sets while maintaining voxel destructibility [S1]. It differs from its parent, Orthographic Voxel Sprite Rendering, by projecting the voxel as a flat 2D square rather than a full 3D sprite billboard, sacrificing volumetric depth cues for rendering speed and simplicity [S1].

## How it works
**Inputs and outputs:**
*   **Input:** A list of voxels, each defined by spatial coordinates $(x, y, z)$ and a color value.
*   **Input:** Screen dimensions (`frame_buffer_width`, `frame_buffer_height`) and center offsets (`half_screen_width`, `half_screen_height`).
*   **Input:** An empirical parameter `SIZE_FACTOR` (square size).
*   **Input:** A depth buffer (`zBuffer`) initialized to maximum depth or infinity.
*   **Output:** A 2D framebuffer (`frameBuffer`) containing the final pixel colors.

**Data structures:**
*   `voxel`: A struct or object with fields `x`, `y`, `z` (floats), and `color` (e.g., RGBA or integer).
*   `frameBuffer`: A 1D array of size `frame_buffer_width * frame_buffer_height`.
*   `zBuffer`: A 1D array of size `frame_buffer_width * frame_buffer_height`, storing depth values.

**Parameters:**
*   `SIZE_FACTOR`: An empirical constant controlling the mapped square size. It is usually constant per frame. If the model is far, use a smaller number to prevent overlap; if close, use a larger number to prevent gaps [S1].

**Steps:**
1.  For each voxel in the input set:
    a. Calculate the inverse depth: $per\_z = 1.0 / voxel.z$.
    b. Determine the 2D screen coordinates of the square's corners using the projection formulas:
       $$xs1 = ((voxel.y - SIZE\_FACTOR) \times per\_z) + half\_screen\_width$$
       $$ys1 = -((voxel.x - SIZE\_FACTOR) \times per\_z) + half\_screen\_height$$
       $$xs2 = ((voxel.y + SIZE\_FACTOR) \times per\_z) + half\_screen\_width$$
       $$ys2 = -((voxel.x + SIZE\_FACTOR) \times per\_z) + half\_screen\_height$$
       Where $xs1, ys1$ is the top-left corner and $xs2, ys2$ is the bottom-right corner [S1].
    c. Rasterize the square defined by $[xs1, xs2]$ and $[ys1, ys2]$:
       i. Iterate $i$ from $\lfloor xs1 \rfloor$ to $\lceil xs2 \rceil$.
       ii. Iterate $j$ from $\lfloor ys1 \rfloor$ to $\lceil ys2 \rceil$.
       iii. Calculate the framebuffer index: $index = i \times frame\_buffer\_width + j$.
       iv. Skip the pixel if it is out of bounds: if $j > frame\_buffer\_width$ or $i > frame\_buffer\_height$ or $j < 0$ or $i < 0$, continue to the next iteration [S1].
       v. Perform depth testing: if $voxel.z < zBuffer[index]$, update the buffers:
           $$zBuffer[index] = voxel.z$$
           $$frameBuffer[index] = voxel.color$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_square_based_voxels(voxels, frame_buffer_width, frame_buffer_height, half_screen_width, half_screen_height, size_factor):
    # Initialize buffers
    # Note: zBuffer should be initialized to a value representing "infinity" or max float
    z_buffer = [float('inf')] * (frame_buffer_width * frame_buffer_height)
    frame_buffer = [None] * (frame_buffer_width * frame_buffer_height)

    for voxel in voxels:
        # 1. Calculate inverse depth
        per_z = 1.0 / voxel.z

        # 2. Calculate square corners
        # xs1, ys1 is top-left; xs2, ys2 is bottom-right
        xs1 = ((voxel.y - size_factor) * per_z) + half_screen_width
        ys1 = -((voxel.x - size_factor) * per_z) + half_screen_height
        xs2 = ((voxel.y + size_factor) * per_z) + half_screen_width
        ys2 = -((voxel.x + size_factor) * per_z) + half_screen_height

        # 3. Rasterize the square
        # Determine integer bounds for iteration
        x_start = int(xs1)
        x_end = int(xs2) + 1 # +1 to include the right edge if it falls on an integer
        y_start = int(ys1)
        y_end = int(ys2) + 1 # +1 to include the bottom edge

        for i in range(x_start, x_end):
            for j in range(y_start, y_end):
                # Boundary check
                if j < 0 or i < 0 or j >= frame_buffer_width or i >= frame_buffer_height:
                    continue

                index = i * frame_buffer_width + j

                # Depth test and update
                if voxel.z < z_buffer[index]:
                    z_buffer[index] = voxel.z
                    frame_buffer[index] = voxel.color

    return frame_buffer
```

**Edge cases and pitfalls:**
*   **Gaps and Overlap:** If the model is close to the view plane and `SIZE_FACTOR` is too small, gaps appear between mapped voxels. If the model is far and `SIZE_FACTOR` is too large, squares overlap excessively [S1].
*   **Back-to-Front Rendering:** If voxels are rendered back-to-front, the z-buffer cannot reject non-visible pixels, causing significant overdraw and performance drops [S1].
*   **Division by Zero:** If a voxel has $z=0$, the calculation of $per\_z$ results in infinity. The source does not specify a guard for this, but it is a necessary implementation detail [G].

## Operating characteristics
*   **Time Complexity:** $O(V \times A)$, where $V$ is the number of voxels and $A$ is the area of the mapped square in pixels. The double iteration loop makes it computationally intensive [S1].
*   **Memory Complexity:** $O(W \times H)$ for the framebuffer and z-buffer, where $W$ and $H$ are screen dimensions.
*   **Performance:** On a horse model (49,152 voxels), performance was 355 FPS. On a medkit model (1,548,288 voxels), performance dropped to 63 FPS without optimization [S1].
*   **Strengths:** Real-time performance for small voxel sets; maintains voxel destructibility; avoids GPU load of dense polygons [S1].
*   **Limitations:** Visual compromises (reduced shading/shadows); significant performance degradation with large voxel counts or back-to-front rendering order; requires careful tuning of `SIZE_FACTOR` [S1].
*   **When to use:** 2D computer games where 3D voxel units rotate or animate, requiring a 2D characteristic on screen but stored in 3D format [S1].
*   **When not to use:** Large voxel sets or scenes requiring high visual fidelity with shadows and complex lighting [S1].

## Background
Square-based voxel rendering was introduced by Péter Mileff and Judit Dudra in their paper "Simplified voxel based visualization" [S1]. The method was developed to address the limitations of existing voxel rendering techniques in the context of 2D computer games that feature 3D voxel units. The authors noted that while cube-based approaches could work, very small cubes would create dense sets that were computationally expensive, and larger cubes would not fit the desired 2D aesthetic. The simplified square-based approach was proposed as a solution to efficiently render small models in real-time, particularly for interactive elements like pickable items or animated units [S1].

## Sources
[S1] Simplified voxel based visualization, https://ojs.uni-miskolc.hu/index.php/psaie/article/view/3824, fetched 2026-10-31
