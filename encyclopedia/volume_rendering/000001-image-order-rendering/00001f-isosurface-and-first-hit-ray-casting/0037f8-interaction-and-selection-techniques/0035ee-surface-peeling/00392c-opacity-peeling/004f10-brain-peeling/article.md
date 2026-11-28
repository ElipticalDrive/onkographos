# Brain Peeling

Also called: Brain Peeling Cutting Tool

## Overview
Brain peeling is a view-independent volume visualization technique designed to reveal interior structures located in the vicinity of the brain surface by removing brain tissue step-by-step in the negative normal direction [S1]. Unlike its sibling [Opacity Peeling](8428-opacity-peeling.md), brain peeling relies on a precomputed distance field rather than view-dependent occlusion culling, requiring fewer parameters and offering a more reliable exploration of surface-adjacent features like vessels and pathological lesions [S1].

## How it works
**Inputs:**
1. A binary brain mask volume $M$ (1 for brain tissue, 0 for background) [S1].
2. A scalar volume dataset $V$ (e.g., MRI intensity values) [S1].
3. A user-defined distance threshold $d_{thresh}$ representing the depth of the peel [S1].

**Outputs:**
A rendered image where brain tissue within distance $d_{thresh}$ of the surface is removed, revealing underlying structures [S1].

**Data Structures:**
*   **Distance Field Volume ($D$):** A 3D grid of the same dimensions as $M$ and $V$, where each voxel stores the distance to the brain surface [S1].
*   **Brain Mask ($M$):** Binary volume indicating brain presence [S1].

**Algorithm Steps:**

1.  **Compute Distance Field:**
    Calculate $D$ from the brain mask $M$ using a simple diffusion filter or distance transformation [S1].
    $$ D(x,y,z) = \text{distance\_transform}(M) $$
    Where the distance transform computes the Euclidean distance to the nearest surface voxel [G].

2.  **Shader Evaluation (Per Voxel):**
    For each voxel $(x,y,z)$ in the volume during ray casting or texture-based rendering:
    a. Retrieve the distance value $d = D(x,y,z)$ [S1].
    b. Retrieve the mask value $m = M(x,y,z)$ [S1].
    c. Discard voxels whose distance is lower than the threshold encoding the peel depth [S1].
    $$ \text{discard} = (m > 0) \land (d < d_{thresh}) $$
    The source specifies discarding voxels when the mask intensity is lower than the threshold encoding the distance, which logically corresponds to discarding brain voxels with $D < d_{thresh}$ [S1].

3.  **Alpha Blending for Smoothness:**
    To avoid hard edges at the peel boundary, blend alpha values in a small range in the normal direction [S1].
    The source does not provide the exact blending function. A standard implementation uses a smoothstep or linear interpolation over a small transition width $\epsilon$ around $d_{thresh}$ [G].
    $$ \alpha_{final} = \alpha_{original} \times \text{smoothstep}(d_{thresh} - \epsilon, d_{thresh}, d) $$
    If $d < d_{thresh} - \epsilon$, $\alpha_{final} = 0$. If $d > d_{thresh}$, $\alpha_{final} = \alpha_{original}$ [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_alpha(distance, threshold, epsilon):
    if distance < threshold - epsilon:
        return 0.0
    elif distance > threshold:
        return 1.0
    else:
        return (distance - (threshold - epsilon)) / epsilon

def render_brain_peeling(volume, distance_field, threshold, epsilon=1.0):
    # volume: 3D array of intensities
    # distance_field: 3D array of distances to surface
    # threshold: float, peel depth
    # epsilon: float, blending width
    # Returns: 2D array of accumulated colors
    height, width, depth = volume.shape
    result = []
    for y in range(height):
        for x in range(width):
            alpha_accum = 0.0
            color_accum = [0.0, 0.0, 0.0]
            for z in range(depth):
                d = distance_field[y, x, z]
                alpha = compute_alpha(d, threshold, epsilon)
                if alpha == 0.0:
                    continue
                c = volume[y, x, z]
                weight = alpha * (1.0 - alpha_accum)
                color_accum = [c[i] * weight + color_accum[i] * (1.0 - weight) for i in range(3)]
                alpha_accum += weight
                if alpha_accum >= 1.0:
                    break
            result.append(color_accum)
    return result
```

**Parameters:**
*   $d_{thresh}$: The distance threshold for peeling. Not given in the sources.
*   $\epsilon$: The width of the blending region for smooth visualization. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The accuracy of the peel depends on the precision of the distance field computation [G].
*   **Boundaries:** The technique assumes a robust skull stripping method has been applied to generate the initial brain mask [S1].

## Subtopics

### Automatic Axis-Aligned Cutting Tool
A complementary technique that divides the volume into eight octants and hides those between the viewpoint and the Region of Interest (ROI) [S1]. It is view-dependent and used for coarse exploration, whereas brain peeling is view-independent and used for fine surface exploration [S1].

### Opacity Peeling
The parent technique that uses view-dependent occlusion to reveal hidden structures [G]. Brain peeling is an alternative that is view-independent and requires fewer parameters [S1].

## Operating characteristics
*   **Time Complexity:** The distance field computation is a one-time pre-processing cost [G]. Rendering time incurs minimal performance loss compared to manual clipping, primarily due to the additional texture lookup for the distance field [S1].
*   **Memory Complexity:** Not given in the sources [G].
*   **Measured Performance:** On a Mac Pro 2.66 GHz with 4 GB RAM and ATI Radeon X1900, rendering at 512x512 viewport with 205x225x158 data achieves 7 fps [S1]. The rendering of two additional data sets (T1 ce, FLAIR) reduced the performance to 12 fps because of the supplemental texture lookups [S1].
*   **Strengths:** View-independent, intuitive exploration of surface-adjacent structures, fewer parameters than opacity peeling, reliable for medical experts [S1].
*   **Limitations:** Requires a high-quality brain mask (skull stripping) as a prerequisite [S1]. Not suitable for exploring deep interior structures far from the surface [S1].

## Background
Brain peeling was introduced by Rezk-Salama and Kolb in 2008 as part of a framework for high-quality multimodal volume visualization of intracerebral pathological tissue [S1]. It was inspired by opacity peeling [RSK06] and the work of Tappenbeck et al. [TPD06] [S1]. The technique is specifically designed for neuro-visualization, allowing medical experts to intuitively explore structures like vessels and tumors located near the brain surface, which are difficult to examine with standard cutting planes [S1].

## Sources
[S1] High-Quality Multimodal Volume Visualization of Intracerebral Pathological Tissue, https://doi.org/10.2312/vcbm/vcbm08/167-176, fetched 2026-10-26
