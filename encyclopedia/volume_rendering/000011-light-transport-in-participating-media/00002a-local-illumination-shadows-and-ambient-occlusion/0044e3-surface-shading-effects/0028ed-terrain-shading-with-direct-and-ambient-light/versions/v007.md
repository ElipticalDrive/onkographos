# Terrain Shading with Direct and Ambient Light

Also called: Hemispheric Ambient Occlusion Shading

## Overview
This method computes the final color of a terrain surface point by combining surface albedo, direct sunlight (modulated by shadow and cloud shadow), and hemispheric ambient light (sky and ground bounce modulated by ambient occlusion) [S1, unofficial]. It distinguishes itself from siblings like the Powder Sugar Effect by explicitly separating direct illumination from a two-component hemispheric ambient model, and from Terrain Shader Lighting and Effects by relying on a specific baked normal map and bias-based shadow tracing rather than generic lighting parameters [S1, unofficial].

## How it works
**Inputs:**
*   Surface position $\mathbf{p} = (x, y, z)$ [S1, unofficial].
*   Surface normal $\mathbf{n} = (n_x, n_y, n_z)$, normalized [S1, unofficial].
*   Albedo color $\mathbf{C}_{albedo}$ [S1, unofficial].
*   Sun direction $\mathbf{l}_{sun}$ and sun radiance $\mathbf{C}_{sun}$ [S1, unofficial].
*   Sky ambient color $\mathbf{C}_{sky}$ and ground ambient color $\mathbf{C}_{ground}$ [S1, unofficial].
*   Cloud shadow factor $S_{cloud}$ at $\mathbf{p}$ [S1, unofficial].
*   Terrain grid cell size (for bias calculation) [S1, unofficial].

**Outputs:**
*   Final color $\mathbf{C}_{out}$ [S1, unofficial].

**Data Structures:**
*   The normal $\mathbf{n}$ is typically baked into a texture and interpolated across the cell, rather than computed via finite differences, to avoid facet artifacts on coarse heightfields [S1, unofficial].

**Steps:**

1.  **Direct Light Calculation:**
    a.  Compute the dot product between the normal and sun direction:
        $$ N \cdot L = n_x l_x + n_y l_y + n_z l_z $$
    b.  Calculate a bias to offset the shadow trace origin, preventing self-shadowing artifacts on coarse grids:
        $$ \text{bias} = \text{cell\_size} \times 0.5 + 0.5 $$
        [S1, unofficial]
    c.  Trace a shadow ray from the biased position:
        $$ \mathbf{p}_{start} = \mathbf{p} + \mathbf{n} \times \text{bias} + \text{jitter} $$
        [S1, unofficial]
        The jitter is applied to reduce aliasing. The shadow factor $S_{shadow}$ is determined by whether this ray hits terrain geometry.
    d.  Compute the direct light scale:
        $$ \text{lightScale} = \max(0, N \cdot L) \times S_{shadow} \times S_{cloud} $$
        [S1, unofficial]
    e.  Compute the direct color contribution:
        $$ \mathbf{C}_{direct} = \mathbf{C}_{sun} \times \text{lightScale} $$
        [S1, unofficial]

2.  **Ambient Occlusion Calculation:**
    a.  Trace an ambient occlusion ray or compute a precomputed occlusion value $AO$ based on local density/geometry around $\mathbf{p}$ [S1, unofficial].
    b.  [G] If not precomputed, this typically involves sampling the hemisphere or using a simplified depth-based approximation.

3.  **Hemispheric Ambient Calculation:**
    a.  Compute the sky factor based on the normal's Y component:
        $$ \text{sky} = 0.5 + 0.5 \times n_y $$
        [S1, unofficial]
    b.  Interpolate between sky and ground ambient colors:
        $$ \mathbf{C}_{ambient\_base} = \mathbf{C}_{sky} \times \text{sky} + \mathbf{C}_{ground} \times (1 - \text{sky}) $$
        [S1, unofficial]
    c.  Apply occlusion:
        $$ \mathbf{C}_{ambient} = \mathbf{C}_{ambient\_base} \times AO $$
        [S1, unofficial]

4.  **Final Color Composition:**
    a.  Combine direct and ambient contributions, modulated by albedo:
        $$ \mathbf{C}_{out} = \mathbf{C}_{albedo} \times (\mathbf{C}_{direct} + \mathbf{C}_{ambient}) $$
        [S1, unofficial]

**Parameters:**
*   `cell_size`: The resolution of the terrain grid [S1, unofficial].
*   `sky`, `ground`: Ambient color vectors [S1, unofficial].
*   `sunDirection`, `sunRadiance`: Light source properties [S1, unofficial].

**Pitfalls:**
*   **Self-shadowing:** The shadow origin must be biased clear of the surface. A bias of `field.cell * 0.5 + 0.5` is recommended to prevent artifacts on coarse heightfields [S1, unofficial].
*   **Normal Quality:** Using finite differences for normals on a bilinear heightfield results in piecewise constant normals, causing visible facet edges. Baked normal maps interpolated across cells are required for smooth shading [S1, unofficial].
*   **Masking Inputs:** Material masks (e.g., for slope or moisture) require smooth inputs. Per-cell slope thresholds create single-cell patches, and normalized fractal noise clusters tightly, leading to thin contour stripes. Slope fields should be blurred, and moisture fields expanded around their midpoint before driving masks [S1, unofficial].

## Subtopics

### Height Shading
A global luma darkening effect relative to the map ceiling, distinct from the local illumination of the main method. It darkens terrain based on depth from the map ceiling using an exponential contrast curve, applied in-place to color buffers.
*   **Definition:** Darkens terrain based on vertical distance from the map top.
*   **When to choose:** When a global atmospheric depth cue or fog-like darkening is desired, rather than local shadowing.
*   **Link:** [Height Shading](9401-height-shading.md) (hypothetical link for variant)

### Screen-Space Bent Cones
A screen-space method to compute directional occlusion, differing from the hemispheric approach by estimating a visibility cone axis and apex angle.
*   **Definition:** Computes directional occlusion using Z-Buffer data to estimate screen-space bent cones, which are circular apertures.
*   **When to choose:** When directional occlusion is needed for better image quality than standard AO, and screen-space data is available.
*   **Link:** [Screen-Space Bent Cones](9401-screen-space-bent-cones.md) (hypothetical link for variant)

## Operating characteristics
*   **Time Complexity:** Dominated by the shadow trace and ambient occlusion trace. If AO is precomputed, the cost is proportional to the shadow ray length and sampling. If AO is traced per-pixel, it adds significant cost.
*   **Memory Complexity:** Low, requiring storage for albedo, baked normals, and potentially precomputed AO maps [S1, unofficial].
*   **Strengths:** Provides realistic terrain shading with distinct sky/ground ambient contributions and handles cloud shadows [S1, unofficial].
*   **Limitations:** Relies on baked normals for quality; finite differences cause artifacts [S1, unofficial]. Does not handle interreflections or small area lights well [S3, unofficial].
*   **When to use:** For static or semi-static terrain rendering where baked data is feasible [S1, unofficial].
*   **When not:** For dynamic scenes requiring real-time normal updates without baking, or when high-fidelity global illumination is required [S3, unofficial].

## Background
The method originates from the `theme1` project by mingderwang, which implements a procedural terrain renderer with specific lighting themes [S1, unofficial]. It addresses the need for realistic terrain appearance in procedural worlds, distinguishing itself by using baked normals and a specific hemispheric ambient model [S1, unofficial]. The concept of hemispheric ambient occlusion is also related to broader techniques in screen-space lighting, such as bent cones, which aim to improve directional occlusion quality [S3, unofficial].

## Sources
[S1] mingderwang/theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
[S2] herbal1st/pyvorengi-sdk-demo, https://github.com/herbal1st/pyvorengi-sdk-demo, fetched 2026-10-27 (unofficial)
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
