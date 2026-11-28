# Octree Light Propagation Volumes

Also called: Light Propagation Volumes with Octrees, Octree LPV

## Overview
Octree Light Propagation Volumes (OLPV) is a real-time global illumination technique that represents Light Propagation Volumes using a full octree data structure to allow light from regular point light sources to be injected, enabling faster propagation and rendering by performing propagation at lower resolutions for distant areas [S1, unofficial][S2, unofficial][S4, unofficial]. It sets itself apart from its sibling [Cascaded Light Propagation Volumes](4333-cascaded-light-propagation-volumes.md) by using a single hierarchical volume covering the same world space with varying resolutions, prioritizing proximity to scene geometry and reflected indirect light rather than viewer proximity [S1, unofficial][S2, unofficial][S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene geometry, omnidirectional point light sources, Reflective Shadow Maps (RSMs) [S1, unofficial][S2, unofficial][S4, unofficial].
*   **Outputs:** A merged uniform Light Propagation Volume (LPV) $M$ used for rendering indirect illumination [S1, unofficial][S2, unofficial][S4, unofficial].

**Data Structures:**
1.  **Octree Levels ($O_i$):** A full octree with $l = \log_2(n) + 1$ levels, where $n$ is the base resolution (a power of two) [S1, unofficial][S2, unofficial][S4, unofficial]. Level $i=0$ has size $n \times n \times n$ [S1, unofficial][S2, unofficial][S4, unofficial]. Level $i$ has size $(n/2^i) \times (n/2^i) \times (n/2^i)$ [S1, unofficial][S2, unofficial][S4, unofficial]. Memory is laid out linearly, allocating all nodes down to a pre-defined level, allowing index/address computation for any node [S1, unofficial][S2, unofficial][S4, unofficial].
2.  **Index Volume ($I_i$):** A hierarchy of structures $I_i$ for $i=0, \dots, l-1$ that store integer indices indicating which octree level to sample for a given world space position [S1, unofficial][S2, unofficial][S4, unofficial]. Layout: Same 3D grid layout as the data storage. Size: $I_0$ has size $n \times n \times n$ [S1, unofficial][S2, unofficial][S4, unofficial]. Content: Integer indices $0, \dots, l-1$ [S1, unofficial][S2, unofficial][S4, unofficial]. Initialization: All elements $I_i(x,y,z)$ are initialized to $l-1$ [S1, unofficial][S2, unofficial][S4, unofficial]. Purpose: Stores the octree level to sample for a given world space position. Lower values indicate proximity to geometry/reflected light [S1, unofficial][S2, unofficial][S4, unofficial].

**Numbered Steps:**

1.  **Light Injection:** Render the scene from each light source into six Reflective Shadow Maps (RSMs) [S1, unofficial][S2, unofficial][S4, unofficial]. Convert the texels in the RSMs into Virtual Point Lights (VPLs) [S1, unofficial][S2, unofficial][S4, unofficial]. Inject the contributions of the VPLs into the octree level $O_0$ using the same method as the original LPV technique (two-band spherical harmonics) [S1, unofficial][S2, unofficial][S4, unofficial].
2.  **Downsampling:** For each level $i$ from $1$ to $l-1$, populate element $O_i(x,y,z)$ by averaging the corresponding eight elements from the previous level $O_{i-1}$ [S1, unofficial][S2, unofficial][S4, unofficial]. This step is performed after and without interaction with the injection step [S1, unofficial][S2, unofficial][S4, unofficial].
    $$O_i(x, y, z) = \frac{1}{8} \sum_{\hat{x}=2x}^{2x+1} \sum_{\hat{y}=2y}^{2y+1} \sum_{\hat{z}=2z}^{2z+1} O_{i-1}(\hat{x}, \hat{y}, \hat{z})$$
    The indices $\hat{x}, \hat{y}, \hat{z}$ range over the children of $(x,y,z)$ in the previous level [S1, unofficial][S2, unofficial][S4, unofficial].
3.  **Index Volume Population:** The index volume $I$ is populated to reflect the octree structure [S1, unofficial]. Generally, lower values are assigned close to geometry and higher values in empty areas [S1, unofficial]. The specific algorithm for populating $I$ is not detailed in the provided notes, but it determines the sampling level [S1, unofficial].
4.  **Propagation:** Perform light propagation individually on each level of the octree using an iterative process where each element receives contributions from surrounding elements [S1, unofficial][S2, unofficial][S4, unofficial]. Use the light propagation scheme from Kaplanyan and Dachsbacher [KD10], which includes maintaining a separate grid containing the geometry of the scene [S1, unofficial][S2, unofficial][S4, unofficial]. Light on lower levels (coarser resolution) propagates further each iteration, achieving coverage of the entire scene with fewer iterations [S1, unofficial][S2, unofficial][S4, unofficial].
    *   Number of iterations $k=4$ [S1, unofficial][S2, unofficial][S4, unofficial].
    *   Propagation factor $p=3$ [S1, unofficial][S2, unofficial][S4, unofficial].
5.  **Merging:** After propagation, sample the octree using the sampling function $S(x,y,z)$ to get values, then merge them into a single traditional uniform LPV $M$ [S1, unofficial][S2, unofficial][S4, unofficial].
    $$M(x, y, z) = S(x, y, z)$$
    This step is necessary because directly sampling all levels of the octree and index volume during rendering is impractical [S1, unofficial][S2, unofficial][S4, unofficial].
6.  **Rendering:** Render the scene using the merged single LPV $M$ [S1, unofficial][S2, unofficial][S4, unofficial]. Use the exact same rendering step as in the traditional LPV technique [S1, unofficial][S2, unofficial][S4, unofficial]. The volume is sampled only once in each $4 \times 4$ region to allow for high quality shadows while maintaining reasonable performance [S1, unofficial][S2, unofficial][S4, unofficial].

**Sampling Function:**
When sampling the octree (for merging or debugging), the world position is mapped to an element in the index volume [S2, unofficial]. The value in the index volume is used to do a second mapping into the correct octree level [S1, unofficial]. The complication is that there are $l$ index volume elements that could be picked for each world space position [S1, unofficial]. To solve this, the element containing the minimum value is used [S1, unofficial].

$$I(x, y, z) = \min_i \left( I_i \left( \left\lfloor \frac{x}{2^i} \right\rfloor, \left\lfloor \frac{y}{2^i} \right\rfloor, \left\lfloor \frac{z}{2^i} \right\rfloor \right) \right)$$

The resulting sampling in the octree corresponds to:
$$S(x, y, z) = O_{I(x,y,z)} \left( \left\lfloor \frac{x}{2^{I(x,y,z)}} \right\rfloor, \left\lfloor \frac{y}{2^{I(x,y,z)}} \right\rfloor, \left\lfloor \frac{z}{2^{I(x,y,z)}} \right\rfloor \right)$$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def create_grid(dim):
    return [[[0.0] * dim for _ in range(dim)] for _ in range(dim)]

def copy_grid(grid):
    dim = len(grid)
    return [[[grid[i][j][k] for k in range(dim)] for j in range(dim)] for i in range(dim)]

def octree_light_propagation_volumes(scene, lights, n, k=4, p=3):
    l = int(math.log2(n)) + 1
    
    # 1. Initialize Octree Levels and Index Volume
    O = [create_grid(n // (2**i)) for i in range(l)]
    I = [create_grid(n) for _ in range(l)]
    
    # Initialize Index Volume to max level
    for i in range(l):
        for x in range(n):
            for y in range(n):
                for z in range(n):
                    I[i][x][y][z] = l - 1
        
    # 2. Light Injection
    # [G] Renders scene from each light into 6 RSMs, converts texels to VPLs.
    # [G] Injects VPL contributions into O[0] using standard LPV injection (2-band SH).
    
    # 3. Downsampling
    for i in range(1, l):
        prev_level = O[i-1]
        curr_level = O[i]
        dim_curr = len(curr_level)
        
        for x in range(dim_curr):
            for y in range(dim_curr):
                for z in range(dim_curr):
                    sum_val = 0.0
                    for dx in range(2):
                        for dy in range(2):
                            for dz in range(2):
                                sum_val += prev_level[2*x + dx][2*y + dy][2*z + dz]
                    curr_level[x][y][z] = sum_val / 8.0

    # 4. Populate Index Volume (I)
    # [G] Algorithm not detailed in sources; typically assigns lower indices near geometry.
    # [G] Populates I based on geometry proximity; lower values near geometry.

    # 5. Propagation
    for i in range(l):
        # [G] Uses Kaplanyan and Dachsbacher [KD10] scheme with geometry occlusion.
        # [G] Standard LPV propagation: each element receives contribution from neighbors
        # scaled by propagation factor p and rotated/scalar adjusted for SH.
        new_grid = copy_grid(O[i])
        # ... propagation logic ...
        O[i] = new_grid

    # 6. Merging
    M = create_grid(n)
    
    for x in range(n):
        for y in range(n):
            for z in range(n):
                min_idx = l - 1
                for i in range(l):
                    idx_i = I[i][x // (2**i)][y // (2**i)][z // (2**i)]
                    if idx_i < min_idx:
                        min_idx = idx_i
                
                level = min_idx
                ox = x // (2**level)
                oy = y // (2**level)
                oz = z // (2**level)
                
                M[x][y][z] = O[level][ox][oy][oz]

    return M
```

## Operating characteristics
*   **Time Complexity:** Propagation is performed on each level individually [S1, unofficial]. Light on lower levels (coarser resolution) propagates further, reducing the total number of iterations required from $k=16$ (uniform LPV) or $k=32$ (theoretical) to $k=4$ [S1, unofficial][S2, unofficial][S4, unofficial]. This increases propagation speed by almost 60% [S1, unofficial][S2, unofficial][S4, unofficial].
*   **Memory Complexity:** Uses a full octree, which is highly inefficient for sparse data but suitable here because larger nodes represent coarse data [S1, unofficial][S2, unofficial][S4, unofficial]. The index volume adds overhead [S1, unofficial][S2, unofficial][S4, unofficial].
*   **Measured Performance:** Renders the Sponza scene at 9 fps, outperforming the original LPV technique by 8% on an NVIDIA GeForce GTX 480 [S1, unofficial][S2, unofficial][S4, unofficial]. NVIDIA's CLPV implementation reaches 60-80 fps with a single directed light source, whereas this implementation uses 18 RSMs [S1, unofficial][S2, unofficial][S4, unofficial].
*   **Strengths:** Faster propagation due to fewer iterations; allows injection of regular point light sources; simpler memory access patterns than sparse pointer-based octrees [S1, unofficial][S2, unofficial][S4, unofficial].
*   **Limitations:** Full octrees are memory inefficient for sparse data; small intensity variations between levels exist but have little visual impact [S1, unofficial][S2, unofficial][S4, unofficial]; sampling all octree levels in the rendering pipeline is impractical, requiring a merge step [S1, unofficial][S2, unofficial][S4, unofficial].
*   **When to use:** When proximity to scene geometry and reflected indirect light is prioritized over viewer proximity, and point light injection is desired [S1, unofficial][S2, unofficial][S4, unofficial].
*   **When not to use:** When memory is extremely constrained and the scene is very sparse, or when viewer-centric cascading is preferred [S1, unofficial][S2, unofficial][S4, unofficial].

## Background
*   **Origin:** Proposed by John David Olovsson and Michael Doggett in 2013 [S1, unofficial][S2, unofficial][S4, unofficial].
*   **History:** Builds on the original Light Propagation Volumes technique by Kaplanyan and Dachsbacher [KD10] and the concept of storing pre-computed indirect illumination using spherical harmonics by Sloan et al. [SKS02] [S1, unofficial]. It contrasts with Crassin et al.'s sparse pointer-based octree technique [S1, unofficial][S2, unofficial][S4, unofficial].
*   **Usage:** Implemented using OpenSceneGraph with osgCompute and CUDA for real-time rendering [S1, unofficial][S2, unofficial][S4, unofficial]. Tested on the Sponza dataset [S1, unofficial][S2, unofficial][S4, unofficial]. Used in NVIDIA's CLPV implementation as a reference for propagation parameters [S1, unofficial][S2, unofficial][S4, unofficial].

## Sources
[S1] Octree Light Propagation Volumes, https://lup.lub.lu.se/record/4002700, fetched 2026-10-26 (unofficial)
[S2] Octree Light Propagation Volumes, https://core.ac.uk/download/83664325.pdf, fetched 2026-10-30 (unofficial)
[S3] Octree Light Propagation Volumes, https://core.ac.uk/download/83664325.pdf, fetched 2026-10-30 (unofficial)
[S4] Octree Light Propagation Volumes, https://core.ac.uk/works/100893913, fetched 2026-11-03 (unofficial)
