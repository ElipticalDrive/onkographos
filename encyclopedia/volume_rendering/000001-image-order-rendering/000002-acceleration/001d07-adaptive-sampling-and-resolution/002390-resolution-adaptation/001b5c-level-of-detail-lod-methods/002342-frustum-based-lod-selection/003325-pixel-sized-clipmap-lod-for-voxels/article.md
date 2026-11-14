# Pixel-Sized Clipmap LOD for Voxels

Also called: Clipmap rings, Pixel-sized LOD

## Overview
Pixel-Sized Clipmap LOD is a level-of-detail strategy where full-resolution voxels are retained only within a radius defined by the camera's pixel footprint, with coarser resolutions arranged in concentric clipmap rings [S1]. It differs from footprint-based methods by strictly tying the LOD boundary to the angular size of a single pixel at the current resolution and field of view, ensuring that the finest detail is never over-sampled relative to the screen [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** Voxel size $s$ (world units per voxel), camera vertical field of view $\theta_{v}$, screen height $H$ (pixels), and the desired number of LOD rings $L$ [S1].
*   **Outputs:** A set of clipmap rings, where ring $i$ contains voxels of size $s \cdot 2^i$ spanning a specific radial distance from the camera [S1].

**Data Structures**
The data is organized into concentric rings around the camera position [S1]. Each ring $i$ (where $i=0$ is the innermost full-resolution ring) stores voxels at resolution $s_i = s \cdot 2^i$ [S1].
*   **Ring $i$ Bounds:** Spans radial distance $[R_{i-1}, R_i]$ where $R_i = R_0 \cdot 2^i$ [S1].
*   **Voxel Count per Ring:** Each ring holds approximately $0.75 \times N_0$ voxels, where $N_0$ is the voxel count in the central disk [S1].
*   **Storage Format:** Voxels can be stored in a compact format (e.g., 3.3 bytes/voxel) or raw format (e.g., 8 bytes/voxel) [S1].

**Algorithm Steps**
1.  **Calculate Pixel Angular Size:** Determine the angular span of one pixel. For a vertical FOV $\theta_{v}$ and screen height $H$:
    $$ \alpha = \frac{\theta_{v}}{H} $$
    Typical value for 1440p at 70° vertical FOV: $\alpha \approx 8.48 \times 10^{-4}$ radians [S1].
2.  **Determine Full-Resolution Radius ($R_0$):** Calculate the distance at which a single voxel of size $s$ subtends exactly one pixel.
    $$ R_0 = \frac{s}{\alpha} \approx 1180 \cdot s $$
    For $s=2$ cm, $R_0 \approx 24$ m. For $s=25$ cm, $R_0 \approx 295$ m [S1].
3.  **Calculate Central Disk Voxels ($N_0$):** Compute the number of voxels required to fill the disk of radius $R_0$ at resolution $s$.
    $$ N_0 = \frac{\pi R_0^2}{s^2} = \pi \left(\frac{1180 \cdot s}{s}\right)^2 = \pi \cdot 1180^2 \approx 4.4 \times 10^6 $$
    If a roughness factor $k=2$ is applied, $N_0 \approx 8.7 \times 10^6$ [S1].
4.  **Define Clipmap Rings:** For each subsequent ring $i$ from 1 to $L$:
    *   **Voxel Size:** $s_i = s \cdot 2^i$.
    *   **Outer Radius:** $R_i = R_0 \cdot 2^i$.
    *   **Voxel Count:** $N_i = 0.75 \times N_0$ [S1].
5.  **Total Resident Voxels:** The total number of voxels resident in memory for $L$ rings is:
    $$ N_{total} = N_0 \cdot (1 + 0.75 \cdot L) $$
    Example: For $L=6$, $N_{total} \approx 4.8 \times 10^7$ [S1].

**Parameters and Typical Values**
*   **LOD Levels ($L$):** To reach distances of 1–5 km from $R_0$:
    *   $s=2$ cm: 5–8 levels [S1].
    *   $s=5$ cm: 4–6 levels [S1].
    *   $s=10$ cm: 3–5 levels [S1].
    *   $s=25$ cm: 2–4 levels [S1].
*   **Memory Footprint:**
    *   6 rings: 160 MB (compact) / 390 MB (raw) [S1].
    *   8 rings: 200 MB (compact) / 500 MB (raw) [S1].
    *   12 rings: 290 MB (compact) / 710 MB (raw) [S1].
*   **High-Quality Headroom:** For path-traced detail at 2 voxels per pixel, multiply resident memory by 4 (approx. 0.6–2.8 GB) [S1].

**Edge Cases and Pitfalls**
*   **Precision:** At 2 cm resolution, a 32-bit integer voxel coordinate spans $\approx 85,900$ km, which is ample [S1]. Float32 positions at 5 km from origin have a step of 0.49 mm, which is sufficient for camera-relative coordinates [S1].
*   **Full-Resolution Region:** The region within $R_0$ must remain full-resolution regardless of pixel size for dynamic elements like edits, debris, fluids, and collisions [S1]. For a 64 m radius per player, this costs 211 MB (compact) at 2 cm resolution [S1].
*   **Memory Scaling:** While the clipmap scales logarithmically, the full-resolution dynamic region scales with the square of the voxel size [S1]. Smaller $s$ significantly increases the cost of the central dynamic region [S1].

## Subtopics

### Camera-Centred Ring LOD Scheme
A Level of Detail scheme covering all-voxel to orbit views using camera-centred rings where each level is twice as coarse as the one inside it, using the same brick format at each mip level [S2, unofficial]. It employs approximately 19–20 levels, ranging from 10 cm resolution out to the Planet-disc Zoom level [S2, unofficial]. Unlike hybrid approaches, it uses no heightfield or mesh fallback, relying entirely on voxel data for all distances [S2, unofficial]. Downsampling is performed such that a coarse voxel is solid if at least 4 of its 8 children are solid, taking the most common solid child's material [S2, unofficial]. Edits propagate up all levels immediately [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The number of LOD levels grows logarithmically with distance [S1]. On-demand generation cost scales with voxel density; for example, 2 cm resolution requires 25× more voxel evaluations than 10 cm resolution for the same area [S1].
*   **Memory Complexity:** Memory usage is bounded by the number of rings $L$ [S1]. Total voxels scale linearly with $L$ ($N_0 \cdot (1 + 0.75 \cdot L)$) [S1].
*   **Performance:** The method fits within 8–12 GB of VRAM, leaving room for path-tracing buffers and lighting caches [S1]. Aokana keeps only ~2–17% of a scene resident with distance LOD, confirming the efficiency of this bounded memory approach [S1].
*   **Strengths:** Bounded memory usage; efficient coverage of large distances; simple ring-based data structure [S1].
*   **Limitations:** Requires significant memory for high-resolution dynamic regions near the camera; quality headroom for high-fidelity rendering (2 voxels/pixel) increases memory demands substantially [S1].

## Background
This method is described in the context of voxel data structures for large-scale environments, specifically in the "bhayanakvoxel" project research documentation [S1]. It is also referenced in discussions regarding voxel data structure and LOD scheme selection for all-voxel to orbit views [S2, unofficial].

## Sources
[S1] voxel-data-structures.md, https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, fetched 2026-10-28
[S2] Choose voxel data structure & LOD scheme #9, https://github.com/theHimanshuShekhar/bhayanakvoxel/issues/9, fetched 2026-10-28 (unofficial)
