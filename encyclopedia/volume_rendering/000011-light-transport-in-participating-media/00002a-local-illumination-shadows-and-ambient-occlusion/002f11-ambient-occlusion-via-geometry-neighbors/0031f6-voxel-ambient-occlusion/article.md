# Voxel Ambient Occlusion

Also called: Voxel AO, computeAO

## Overview
Voxel Ambient Occlusion (Voxel AO) estimates ambient occlusion for a surface point by sampling the solidity of surrounding voxels in a grid relative to the surface normal, simulating light blockage from adjacent geometry [S1, unofficial]. It differs from [Ambient Occlusion via SDF/Geometry](2527-ambient-occlusion-via-sdf-geometry.md) by operating on discrete voxel occupancy grids rather than continuous Signed Distance Fields, and from [JAOCC Shading](3634-jaocc-shading.md) by using explicit neighbor queries instead of precomputed cache structures. It builds on [Ambient Occlusion via Geometry/Neighbors](5936-ambient-occlusion-via-geometry-neighbors.md) by providing a specific implementation for voxelized data.

## How it works
**Inputs and Outputs:**
*   **Input:** Surface hit position $\mathbf{p}_{hit}$ (world space), surface normal $\mathbf{n}$ (world space), voxel grid resolution/scale, voxel grid offset, and a voxel solidity query function `isSolid(pos)` returning 1 if occupied, 0 otherwise.
*   **Output:** A scalar ambient occlusion factor $AO \in [0, 1]$, where 0 is fully occluded and 1 is fully open.

**Data Structures:**
*   **Voxel Grid:** A 3D array or hash map storing binary or density values. For AO, binary solidity (occupied/unoccupied) is typically sufficient [S1, unofficial].
*   **Offset Array:** A 2D array `axes` of 4 elements representing corner offsets in the local plane of the surface: `[(1,1), (-1,1), (-1,-1), (1,-1)]` [S1, unofficial].

**Parameters:**
*   `epsilon`: Small offset to prevent self-intersection, typically $0.01$ [S1, unofficial].
*   `weights`: Coefficients for side and corner contributions. Typical values: side weight $0.4$, corner weight $0.2$, full occlusion weight $0.9$ [S1, unofficial].

**Steps:**
1.  **Base Position Calculation:** Determine the integer voxel coordinate of the hit point, offset by the normal to ensure we are querying the immediate exterior/neighborhood:
    $$ \mathbf{p}_{base} = \lfloor \mathbf{p}_{hit} + \mathbf{n} \cdot \epsilon \rfloor $$
    where $\lfloor \cdot \rfloor$ denotes component-wise floor operation [S1, unofficial].

2.  **Determine Dominant Axis:** Identify the dominant axis of the normal $\mathbf{n}$ to define the local "up" and "right" vectors for the voxel face.
    *   If $|n_x| > 0$: $\mathbf{sideA} = (0, \Delta y, 0)$, $\mathbf{sideB} = (0, 0, \Delta z)$ [S1, unofficial].
    *   Else if $|n_y| > 0$: $\mathbf{sideA} = (\Delta x, 0, 0)$, $\mathbf{sideB} = (0, 0, \Delta z)$ [S1, unofficial].
    *   Else: $\mathbf{sideA} = (\Delta x, 0, 0)$, $\mathbf{sideB} = (0, \Delta y, 0)$ [S1, unofficial].
    *   Here $\Delta x, \Delta y, \Delta z$ are the voxel grid step sizes (or 1 if using integer grid coordinates directly) [S1, unofficial].

3.  **Neighbor Sampling:** Sample the 4 corners of the voxel face adjacent to the hit point. The corners are defined by combinations of $\mathbf{sideA}$ and $\mathbf{sideB}$:
    *   $\mathbf{c}_1 = \mathbf{p}_{base} + \mathbf{sideA} + \mathbf{sideB}$
    *   $\mathbf{c}_2 = \mathbf{p}_{base} - \mathbf{sideA} + \mathbf{sideB}$
    *   $\mathbf{c}_3 = \mathbf{p}_{base} - \mathbf{sideA} - \mathbf{sideB}$
    *   $\mathbf{c}_4 = \mathbf{p}_{base} + \mathbf{sideA} - \mathbf{sideB}$
    *   Query solidity: $s_i = \text{isSolid}(\mathbf{c}_i)$ for $i=1..4$ [S1, unofficial].

4.  **AO Calculation (Deterministic Variant):**
    *   Initialize `aoSum = 0.0` [S1, unofficial].
    *   For each of the 4 sides/corners (or faces defined by the dominant axis), calculate the contribution. A common implementation sums the solidity of the 4 corners and divides by 4.0, or uses a weighted sum [S1, unofficial]:
        $$ AO_{factor} = \frac{1}{4} \sum_{i=1}^{4} s_i $$
    *   Alternatively, a more nuanced approach checks pairs of sides: if both side neighbors are solid, add $0.9$; otherwise, add $s_{side1} \cdot 0.4 + s_{side2} \cdot 0.4 + s_{corner} \cdot 0.2$ [S1, unofficial].
    *   Final AO: $AO = \text{clamp}(1.0 - AO_{factor}, 0.0, 1.0)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_voxel_ao(hit_pos: Vec3, normal: Vec3, voxel_grid: VoxelGrid, voxel_scale: float, voxel_offset: Vec3) -> float:
    # 1. Calculate base voxel position
    # Offset by normal to avoid self-intersection
    epsilon = 0.01
    base_pos_world = hit_pos + normal * epsilon
    base_pos_grid = floor((base_pos_world - voxel_offset) / voxel_scale)
    
    # 2. Determine side vectors based on dominant normal axis
    # Assuming voxel_scale is uniform for simplicity, offsets are 1 in grid coords
    if abs(normal.x) > 0.0:
        side_a = Vec3(0, 1, 0)
        side_b = Vec3(0, 0, 1)
    elif abs(normal.y) > 0.0:
        side_a = Vec3(1, 0, 0)
        side_b = Vec3(0, 0, 1)
    else:
        side_a = Vec3(1, 0, 0)
        side_b = Vec3(0, 1, 0)
        
    # 3. Sample 4 corners
    corners = [
        base_pos_grid + side_a + side_b,
        base_pos_grid - side_a + side_b,
        base_pos_grid - side_a - side_b,
        base_pos_grid + side_a - side_b
    ]
    
    # 4. Query solidity and sum
    ao_sum = 0.0
    for corner_pos in corners:
        # isSolid returns 1.0 if occupied, 0.0 otherwise
        solidity = voxel_grid.is_solid(corner_pos)
        ao_sum += solidity
        
    # 5. Calculate final AO
    # Simple average of corner solidity
    ao_factor = ao_sum / 4.0
    
    # Invert: 0 solidity (empty) -> 1.0 AO (bright), 4 solidity (full) -> 0.0 AO (dark)
    ao = 1.0 - ao_factor
    
    return clamp(ao, 0.0, 1.0)
```

## Subtopics

### Stochastic Hemisphere Sampling for Voxel AO
A stochastic method to estimate ambient occlusion by casting rays into a hemisphere oriented around the surface normal, using cosine-weighted sampling [S1, unofficial]. It differs from the deterministic corner sampling by providing smoother results at the cost of higher computational load and potential noise. Choose this when high-quality soft shadows are needed and deterministic artifacts are unacceptable.
*   **Steps:**
    1.  Loop $i$ from $0$ to `maxSamples`.
    2.  Calculate seed: `seed = seedBase + f32(i) * (13.37 * 10.2)` [S1, unofficial].
    3.  Generate two random numbers $u_1, u_2$ from seed.
    4.  Sample local direction using cosine-weighted hemisphere sampling.
    5.  Transform local direction to world space using tangent (T), bitangent (B), and normal (N) basis.
    6.  Perform a voxel mesh query (ray march) from `hitPosition + normal * epsilon`.
    7.  If a hit is found, increment hit counter.
    8.  Calculate `occlusionFactor = 1.0 - (hit / maxSamples)` [S1, unofficial].
*   **Parameters:** `maxSamples` = 8; `seedBase` = `fragCoord.x * fragCoord.y + 0.1234 + time` [S1, unofficial].
*   **Pitfall:** Epsilon offset (0.01) is critical to prevent self-intersection [S1, unofficial].

### Voxel Ambient Occlusion via Axis-Aligned Corner Sampling
An ambient occlusion technique for voxel scenes that samples the 4 corners of the voxel face adjacent to the hit point, weighted by the surface normal's dominant axis [S1, unofficial]. It is a simplified version of the general Voxel AO, strictly adhering to axis-aligned checks. Choose this for maximum performance on simple voxel grids where normal alignment is predictable.
*   **Steps:**
    1.  Calculate `basePos = floor(hitPos + normal * 0.01)` [S1, unofficial].
    2.  Define 4 corner offsets: `(1,1), (-1,1), (-1,-1), (1,-1)` [S1, unofficial].
    3.  Based on dominant normal axis, set `sideA` and `sideB` vectors [S1, unofficial].
    4.  Sum the solidity of the 4 corners.
    5.  Divide by 4.0 to get AO factor [S1, unofficial].

### Voxel-Neighborhood Ambient Occlusion
A method that converts vertex world-space positions to voxel grid coordinates and queries the neighborhood at `(vx, vy, vz)` and the vertex normal [S3, unofficial]. It is designed for vertex-based shading rather than ray-casted surface shading. Choose this for static geometry where AO can be precomputed or computed per-vertex.
*   **Steps:**
    1.  Convert vertex position to voxel grid coordinates: `vx, vy, vz = floor((pos - offset) / scale)` [S3, unofficial].
    2.  Query voxel neighborhood at `(vx, vy, vz)` using the vertex normal [S3, unofficial].
    3.  Modulate vertex RGB color by multiplying each channel by the computed AO factor [S3, unofficial].
*   **Parameters:** `offset` (world-space origin of voxel grid) [S3, unofficial].

### Color Bleeding via Extinction
An approximation of color bleeding where the color appearance of a surface is affected by colored nearby objects, modeled by including the voxel color $C_{RGB}$ in the attenuation integral [S4, unofficial]. It extends standard Voxel AO by incorporating color information from the transfer function. Choose this when color bleeding is required and the voxel grid stores color data.
*   **Formula:**
    $$ I_{AORGB}(s) \approx I_A \cdot e^{-\int_{t \in \mathbb{N}(s)} \frac{\tau(t) C_{RGB}(t)}{|s-t|^2} dt} $$
    where $I_{AORGB}$ is a vector describing intensity per color, $C_{RGB}(t)$ is the color at position $t$ depending on the Transfer Function (TF), and $\tau(t)$ is the extinction coefficient [S4, unofficial].
*   **Steps:**
    1.  Reformulate the AO integral by multiplying the extinction coefficient $\tau(t)$ by the color $C_{RGB}(t)$ before integration [S4, unofficial].
    2.  Use aggregate summation methods for efficient computation [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for deterministic corner sampling (constant number of voxel lookups). $O(N)$ for stochastic sampling, where $N$ is the number of samples [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory, dependent on the voxel grid storage [S1, unofficial].
*   **Performance:** Deterministic Voxel AO is extremely fast, suitable for real-time rendering on mobile or low-end hardware [S1, unofficial]. Stochastic variants are slower but provide smoother results [S1, unofficial].
*   **Strengths:** Simple to implement, works well with existing voxel engines, provides immediate feedback on local geometry density [S1, unofficial].
*   **Limitations:** Discrete nature can lead to aliasing or "blocky" AO artifacts if voxel resolution is low [S1, unofficial]. Does not account for global illumination or indirect bounces beyond the immediate neighborhood [S4, unofficial].
*   **When to use:** Real-time voxel-based games, simple shadowing approximations, precomputed vertex AO [S1, S3, unofficial].
*   **When not to use:** High-fidelity offline rendering requiring soft global illumination, scenes with very low voxel resolution where artifacts are prominent [S1, unofficial].

## Background
Voxel Ambient Occlusion originated with the rise of voxel-based rendering engines, particularly in the context of games like *Minecraft* and subsequent voxel engine implementations. It was popularized by libraries and engines such as DivineVoxelEngine, which provided efficient implementations for real-time AO in voxel spaces [S1, unofficial]. The technique leverages the inherent grid structure of voxels to simplify visibility queries, avoiding the need for complex ray-triangle intersection tests. Extensions like color bleeding via extinction were developed to enhance visual fidelity by incorporating color information from the voxel data [S4, unofficial].

## Sources
[S1] Divine-Star-Software/DivineVoxelEngine, https://github.com/Divine-Star-Software/DivineVoxelEngine, fetched 2026-10-26 (unofficial)
[S2] ria8651/bevy-voxel-engine, https://github.com/ria8651/bevy-voxel-engine, fetched 2026-10-27 (unofficial)
[S3] elyssov/prometheus-engine, https://github.com/elyssov/prometheus-engine, fetched 2026-10-28 (unofficial)
[S4] PDF Extinction-based Shading and Illumination in GPU Volume Ray-Casting - UZH, https://www.ifi.uzh.ch/dam/jcr:ffffffff-9e7f-2af5-ffff-ffffcf9259bb/FastShadows.pdf, fetched 2026-10-30 (unofficial)
