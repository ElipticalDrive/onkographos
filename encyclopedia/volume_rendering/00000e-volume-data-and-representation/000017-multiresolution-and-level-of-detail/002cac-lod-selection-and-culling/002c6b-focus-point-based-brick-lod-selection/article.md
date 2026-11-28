# Focus Point Based Brick LOD Selection

Also called: Adaptive level-of-detail, Focus Point Based LOD Selection, Focus point based LOD, Focus point oracle, Focus-point based adaptive level-of-detail

## Overview
Focus Point Based Brick LOD Selection is a method for determining the resolution level of individual bricks in a multiscale volume representation based solely on the spatial distance between the brick's center and a user-defined focus point [S1, S2]. Unlike error-based methods, it does not consider approximation error or signal frequency, relying exclusively on a "focus point oracle" to assign levels [S1]. This approach allows for arbitrary texture resolutions per brick and significant memory savings, but risks topological artifacts and boundary discontinuities when high-frequency data is present [S1, S2].

## How it works
**Inputs:**
*   A 3D volume dataset [G].
*   A user-defined focus point $P_f$ in world coordinates [S1, S2].
*   A predefined initial brick size (spatial extent) for the finest level [S2].
*   A mapping function (oracle) from distance to LOD level [S1].

**Outputs:**
*   A multiscale brick hierarchy where each brick is assigned a specific resolution level [S2].
*   Adapted brick boundaries if level transitions occur [S2].

**Data Structures:**
*   **Brick:** A spatial partition of the volume. Each brick stores its spatial bounds and its assigned LOD level [G].
*   **Hierarchy:** A collection of bricks at various resolutions. The structure assumes that adjacent bricks may have different resolutions, requiring boundary adaptation [S2].

**Steps:**
1.  **Define Initial Brick Size:** Specify the physical size of the bricks at the finest resolution level before constructing the hierarchy [S2].
2.  **Calculate Distances:** For each brick $B_i$ in the volume, calculate the Euclidean distance $d_i$ from the brick's center $C_i$ to the focus point $P_f$:
    $$ d_i = \| C_i - P_f \| $$
3.  **Determine LOD Level:** Use the focus point oracle to map $d_i$ to a discrete LOD level $L_i$. The oracle typically defines a threshold or function such that bricks closer to $P_f$ receive higher resolution (lower $L_i$) and distant bricks receive lower resolution (higher $L_i$) [S1, S2].
4.  **Adapt Boundaries:** If the hierarchy is modified (e.g., by moving $P_f$), adapt brick boundaries to ensure continuous level transitions. Specifically, if a brick is adjacent to a coarser brick, its boundaries must be adjusted to match the coarser grid [S2].
5.  **Handle Adjacency:** When adapting boundaries, up to 26 neighboring bricks in 3D may need to be considered to maintain consistency across level transitions [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def euclidean_distance(p1, p2):
    return ((p1[0]-p2[0])**2 + (p1[1]-p2[1])**2 + (p1[2]-p2[2])**2)**0.5

def align_to_coarser_grid(bounds, lod_level, base_size):
    # [G] Standard grid alignment logic for brick boundaries
    new_size = base_size * (2 ** lod_level)
    snapped = []
    for i in range(0, 6, 2):
        snapped.append(round(bounds[i] / new_size) * new_size)
    return snapped

def focus_point_lod_selection(volume_dims, focus_point, initial_brick_size, lod_thresholds):
    # Step 1: Generate initial bricks at finest resolution
    # [G] Assumes a regular 3D grid partitioning for the finest level
    num_bricks_x = volume_dims[0] // initial_brick_size
    num_bricks_y = volume_dims[1] // initial_brick_size
    num_bricks_z = volume_dims[2] // initial_brick_size
    bricks = []
    for z in range(num_bricks_z):
        for y in range(num_bricks_y):
            for x in range(num_bricks_x):
                center = [x * initial_brick_size + initial_brick_size/2,
                          y * initial_brick_size + initial_brick_size/2,
                          z * initial_brick_size + initial_brick_size/2]
                brick = {'center': center, 
                         'bounds': [x*initial_brick_size, y*initial_brick_size, z*initial_brick_size,
                                    (x+1)*initial_brick_size, (y+1)*initial_brick_size, (z+1)*initial_brick_size],
                         'lod_level': 0, 'neighbors': []}
                bricks.append(brick)
    bricks_by_coord = {(b['bounds'][0], b['bounds'][1], b['bounds'][2]): b for b in bricks}

    # Step 2 & 3: Calculate distances and determine LOD level
    for brick in bricks:
        distance = euclidean_distance(brick['center'], focus_point)
        # [G] Oracle maps distance to LOD level; lower distance yields higher resolution (lower level index)
        lod_level = 0
        for i, threshold in enumerate(lod_thresholds):
            if distance > threshold:
                lod_level = i + 1
        brick['lod_level'] = lod_level

    # Step 5: Handle Adjacency (build neighbor list)
    for brick in bricks:
        bx, by, bz = brick['bounds'][0], brick['bounds'][1], brick['bounds'][2]
        for dz in [-1, 0, 1]:
            for dy in [-1, 0, 1]:
                for dx in [-1, 0, 1]:
                    if dx == 0 and dy == 0 and dz == 0: continue
                    nx, ny, nz = bx + dx * initial_brick_size, by + dy * initial_brick_size, bz + dz * initial_brick_size
                    if (nx, ny, nz) in bricks_by_coord:
                        brick['neighbors'].append(bricks_by_coord[(nx, ny, nz)])

    # Step 4: Adapt Boundaries
    for brick in bricks:
        max_coarser_lod = brick['lod_level']
        for neighbor in brick['neighbors']:
            if neighbor['lod_level'] > max_coarser_lod:
                max_coarser_lod = neighbor['lod_level']
        if max_coarser_lod > brick['lod_level']:
            brick['bounds'] = align_to_coarser_grid(brick['bounds'], max_coarser_lod, initial_brick_size)
    return bricks
```

**Parameters:**
*   **Focus Point ($P_f$):** The user-defined point in space that determines the area of highest interest [S1, S2].
*   **Initial Brick Size:** The physical dimension of the finest-level bricks, specified in advance [S2].

**Edge Cases and Pitfalls:**
*   **Approximation Error:** The method does not account for approximation error. High-frequency data sampled at a lower resolution can deviate significantly from the original signal [S1, S2].
*   **Topological Artifacts:** Non-topology preserving coarsification can cause structures to disappear due to low-pass filtering, leading to visual artifacts at brick boundaries [S1].
*   **Boundary Discontinuities:** Even if continuity is guaranteed by boundary adaptation, artifacts may still occur if the resampled approximation differs significantly from the original data [S1, S2].

## Subtopics

### Hierarchical Culling & Multi-resolution Approximation
What it is: A method that uses error metrics to select LOD [G]. What sets it apart: It evaluates approximation error rather than relying solely on distance [G]. When to choose: When visual fidelity and topological preservation are critical [G]. [Link](2010-hierarchical-culling-multi-resolution-approximation.md)

### Streaming and Level-of-Detail (LOD) selection
What it is: A technique focusing on data streaming and dynamic LOD [G]. What sets it apart: It handles continuous data flow, whereas Focus Point Based LOD constructs a static hierarchy based on spatial position [G]. When to choose: For interactive applications with streaming data [G]. [Link](3087-streaming-and-level-of-detail-lod-selection.md)

### Choosing Alternative Resolutions
What it is: A framework for general resolution choice strategies [G]. What sets it apart: It evaluates multiple criteria, contrasting with the single-criterion focus point approach [G]. When to choose: When multiple selection criteria need to be balanced [G]. [Link](5314-choosing-alternative-resolutions.md)

### Finding Higher Resolution Levels
What it is: A process for locating finer resolution levels [G]. What sets it apart: It focuses on refinement, whereas Focus Point Based LOD assigns levels based on a pre-defined oracle [G]. When to choose: When refining specific regions is required [G]. [Link](5315-finding-higher-resolution-levels.md)

### Multi-resolution LOD Volume Rendering with Temporal Coherence
What it is: A method incorporating temporal coherence into LOD selection [G]. What sets it apart: It maintains consistency across frames, while Focus Point Based LOD is frame-independent [G]. When to choose: For animated sequences to reduce flickering [G]. [Link](5717-multi-resolution-lod-volume-rendering-with-temporal-coherenc.md)

### Flat blocking multi-resolution volume rendering
What it is: A representation using flat blocking structures [G]. What sets it apart: It operates on 2D slices, whereas Focus Point Based LOD uses 3D bricks with boundary adaptation [G]. When to choose: For 2D slice-based rendering pipelines [G]. [Link](6428-flat-blocking-multi-resolution-volume-rendering.md)

### Level of Detail (LoD) for Large-Scale Scenes
What it is: General LoD techniques for large scenes [G]. What sets it apart: It often involves aggressive culling, whereas Focus Point Based LOD is specific to brick-based multiscale representations [G]. When to choose: For massive datasets requiring extensive culling [G]. [Link](7325-level-of-detail-lod-for-large-scale-scenes.md)

### Level of Detail Calculation
What it is: General calculation methods for LOD [G]. What sets it apart: It often relies on error-based metrics, contrasting with the distance-only calculation here [G]. When to choose: When abstract LOD metrics are needed [G]. [Link](7836-level-of-detail-calculation.md)

### LoD Selection via Projected Width
What it is: A screen-space LOD selection method [G]. What sets it apart: It uses projected width for LOD, unlike the world-space distance metric of Focus Point Based LOD [G]. When to choose: When screen-space accuracy is prioritized over world-space distance [G]. [Link](7837-lod-selection-via-projected-width.md)

### GPU-Based Dynamic LOD Calculation
What it is: A method implementing LOD calculation on the GPU [G]. What sets it apart: It leverages hardware acceleration, whereas Focus Point Based LOD is described as a CPU-side hierarchy construction method [G]. When to choose: For real-time hardware-accelerated rendering [G]. [Link](10845-gpu-based-dynamic-lod-calculation.md)

### Error-based and Statistical Prediction
What it is: A technique using error metrics or statistics [G]. What sets it apart: It quantifies approximation bounds, directly contrasting with the focus point oracle's lack of error consideration [G]. When to choose: When quantitative error bounds are required [G]. [Link](15224-error-based-and-statistical-prediction.md)

## Operating characteristics
*   **Time Complexity:** The selection process is $O(N)$ where $N$ is the number of bricks, as each brick's distance is calculated once. Boundary adaptation may increase complexity due to neighbor checks (up to 26 neighbors per brick in 3D) [S2].
*   **Memory Complexity:** Reduces texture memory usage significantly. For example, in the head data set, memory usage dropped to 64% and 37% of the original at different LOD settings [S1].
*   **Performance:** Reduces the number of texture lookups (e.g., from 69% to 40% of original) [S1].
*   **Strengths:** Allows arbitrary texture resolutions for every brick, which is not possible with techniques requiring strict correspondence between texture tiles and brick geometry [S1, S2]. Simple to implement as it relies only on distance [G].
*   **Limitations:** Does not consider approximation error, leading to potential loss of relevant structures and visual artifacts at brick boundaries, especially with high-frequency data [S1, S2]. Non-topology preserving coarsification can cause structures to disappear [S1].
*   **When to use:** When computational simplicity is prioritized over visual fidelity, or when the data is known to be low-frequency [G].
*   **When not to use:** When high-frequency details are critical, or when topological accuracy is required, as error-based methods are superior in these cases [S1].

## Background
The method was introduced by Weiler et al. in 2000 in the paper "Level-Of-Detail Volume Rendering via 3D Textures" [S1, S2]. It was developed as part of a broader approach to multiresolution 3D texture-based volume rendering, aiming to reduce memory usage and rendering time by adapting brick resolution to the viewer's focus. The approach was noted as having a major drawback compared to error-controlled methods, prompting further research into pyramidal representations and error-controlled ray-casting [S1, S2].

## Sources
[S1] Level-Of-Detail Volume Rendering via 3D Textures, https://publications.sci.utah.edu/publications/weiler00/IEEE-VolVis-2000.pdf, fetched 2026-10-27
[S2] Level-of-Detail Volume Rendering via 3D Textures, https://www.semanticscholar.org/paper/05e7d88387ac7484397e329456535b0c8c665a16, fetched 2026-11-03
