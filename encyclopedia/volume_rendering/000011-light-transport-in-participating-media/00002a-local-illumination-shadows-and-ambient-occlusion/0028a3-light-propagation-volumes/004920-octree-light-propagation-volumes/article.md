# Octree Light Propagation Volumes


## Overview
Octree Light Propagation Volumes (OLPV) is a real-time global illumination technique that represents Light Propagation Volumes using a full octree data structure [S1, unofficial]. It solves the problem of slow light propagation in uniform grids by allowing light from regular point light sources to be injected and propagated at lower resolutions for distant areas, enabling faster rendering [S1, unofficial]. Unlike Cascaded LPV (CLPV), which prioritizes spatial proximity to the viewer using multiple grids of the same resolution, OLPV prioritizes proximity to scene geometry and reflected indirect light using volumes of different resolutions covering the same world space [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene geometry, light sources (omnidirectional point lights), Reflective Shadow Maps (RSMs).
*   **Outputs:** A merged single LPV volume $M$ ready for standard LPV rendering, and the final illuminated image.

**Data Structures:**
1.  **Full Octree Data Structure ($O$):** A linear memory layout allocating all nodes down to a pre-defined level. Each level $i$ is a 3D grid.
    *   Level $i=0$ has size $n \times n \times n$.
    *   Total levels $l = \log_2 n + 1$.
    *   Indices $\hat{x}, \hat{y}, \hat{z}$ range from $0$ to $2^{l-i}-1$ (Note: Source text says $2^{n-i}$ but context implies $2^{l-i}$ or similar power of 2 scaling; standard octree logic dictates level $i$ is $n/2^i$. The source notation $O_i(\hat{x}, \hat{y}, \hat{z})$ uses indices up to $2^{n-i}$ which seems to be a typo for $2^{l-i}$ or similar, but we follow the source's explicit range definition: "indices are defined within the range $\hat{x}, \hat{y}, \hat{z} \in [0, \dots, 2^{n-i}]$" [S1, unofficial]. *Correction*: The source text actually says "indices are defined within the range $\hat{x}, \hat{y}, \hat{z} \in [0, \dots, 2^{n-i}]$" in section 3.1. However, later it says "Level $i=0$ has size $n \times n \times n$" and "total levels $l = \log_2 n + 1$". If $n=32$, $l=6$. Level 0 is $32^3$. Level 1 is $16^3$. The index range for level $i$ should be $0 \dots n/2^i - 1$. The source text likely contains a typo $2^{n-i}$ instead of $2^{l-i}$ or $n/2^i$. We will use the logical definition derived from the downsampling formula which implies a factor of 2 reduction per level).
    *   Element denoted $O_i(\hat{x}, \hat{y}, \hat{z})$.
2.  **Index Volume Hierarchy ($I$):** A set of structures $I_i$ for $i=0 \dots l-1$, laid out same as data storage.
    *   Each $I_i$ is a 3D grid of integers.
    *   Size of $I_0$ is $n \times n \times n$.
    *   Each element $I_i(x,y,z)$ stores an integer index $0 \dots l-1$ representing the octree level to sample.
    *   Originally initialized to $l-1$ (the highest level index) [S1, unofficial].
3.  **Scene Graph:** Used for real-time implementation. Contains a pre-render branch (RSM creation to merging) and a rendering branch [S1, unofficial].

**Parameters:**
*   **Octree Resolution $n$:** A power of two ($n = 2^1, 2^2, \dots$). Defines the size of the highest resolution level (Level 0) as $n \times n \times n$ [S1, unofficial].
*   **Spherical Harmonics Bands:** The first two bands are used to store light flux (color, intensity, direction) [S1, unofficial].
*   **Propagation Iterations $k$:** Value: 4. Further iterations no longer contribute visually to the resulting intensity levels [S1, unofficial].
*   **Propagation Factor $p$:** Value: 3. Based on NVIDIA’s implementation of CLPV [S1, unofficial].
*   **RSM Resolution:** 1024x1024 [S1, unofficial].
*   **Sampling Region:** 4x4. The volume is sampled only once in each 4x4 region to allow for high quality shadows while maintaining reasonable performance [S1, unofficial].

**Numbered Steps:**

1.  **Light Injection:** Render the scene from each light source into six Reflective Shadow Maps (RSMs) [S1, unofficial]. Convert the texels in the RSMs into Virtual Point Lights (VPLs) [S1, unofficial]. Inject the contributions of the VPLs into the octree level $O_0$ in the exact same way as in the original LPV technique [S1, unofficial].

2.  **Downsampling:** For each level $i$ from 1 to $l-1$, populate element $O_i(x,y,z)$ by averaging the corresponding eight elements from the previous level $O_{i-1}$ [S1, unofficial]. The eight elements are $O_{i-1}([2x, 2x+1], [2y, 2y+1], [2z, 2z+1])$ [S1, unofficial]. This step is done after and without interaction with the injection step [S1, unofficial].
    $$ O_i(x, y, z) = \frac{1}{8} \sum_{\hat{x}=2x}^{2x+1} \sum_{\hat{y}=2y}^{2y+1} \sum_{\hat{z}=2z}^{2z+1} O_{i-1}(\hat{x}, \hat{y}, \hat{z}) $$
    [S1, unofficial]

3.  **Propagation:** Propagate light individually on each level of the octree using an iterative process where each element receives contributions from surrounding elements [S1, unofficial]. Use the light propagation scheme from Kaplanyan and Dachsbacher [KD10], which includes maintaining a separate grid containing the geometry of the scene [S1, unofficial]. Light on lower levels propagates further each iteration, achieving coverage of the entire scene with fewer iterations [S1, unofficial]. The number of iterations is $k=4$ [S1, unofficial].

4.  **Merging:** To avoid accessing all octree and index volume levels during rendering, merge the sampled result into a single traditional $n \times n \times n$ LPV $M$ [S1, unofficial].
    $$ M(x, y, z) = S(x, y, z) $$
    [S1, unofficial]

5.  **Rendering:** Render the scene using the merged single LPV $M$. Use the exact same rendering step as in the traditional LPV technique [S1, unofficial].

**Sampling Function:**
During the merge step (or if sampling directly), the sampling function $S(x, y, z)$ determines which octree level to use:
$$ I(x, y, z) = \min_i \left( I_i \left( \lfloor x/2^i \rfloor, \lfloor y/2^i \rfloor, \lfloor z/2^i \rfloor \right) \right) $$
$$ S(x, y, z) = O_{I(x,y,z)} \left( \lfloor x/2^{I(x,y,z)} \rfloor, \lfloor y/2^{I(x,y,z)} \rfloor, \lfloor z/2^{I(x,y,z)} \rfloor \right) $$
[S1, unofficial]
The element with the minimum value in the index volume hierarchy is used to resolve which level to sample [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
n = 32  # Octree resolution (power of 2)
l = int(log2(n)) + 1  # Total levels
k = 4  # Propagation iterations
p = 3  # Propagation factor

# Data Structures
# O[i][x][y][z] stores SH coefficients for level i, coordinates x,y,z
# I[i][x][y][z] stores the level index for level i, coordinates x,y,z
# M[x][y][z] stores the final merged LPV

# 1. Light Injection
# Render RSMs and create VPLs (external step)
# Inject VPLs into O[0] using standard LPV injection method
inject_vpls_into_octree_level_0(O[0], vpls)

# 2. Downsampling
for i in range(1, l):
    for x in range(n // (2**i)):
        for y in range(n // (2**i)):
            for z in range(n // (2**i)):
                # Average 8 children from previous level
                sum_val = 0
                for dx in range(2):
                    for dy in range(2):
                        for dz in range(2):
                            child_x = 2 * x + dx
                            child_y = 2 * y + dy
                            child_z = 2 * z + dz
                            sum_val += O[i-1][child_x][child_y][child_z]
                O[i][x][y][z] = sum_val / 8.0

# 3. Propagation
# Initialize Index Volume
for i in range(l):
    for x in range(n // (2**i)):
        for y in range(n // (2**i)):
            for z in range(n // (2**i)):
                I[i][x][y][z] = l - 1  # Initialize to highest level

# Propagate each level independently
for i in range(l):
    # Use Kaplanyan and Dachsbacher [KD10] propagation scheme
    # Maintain separate geometry grid for occlusion
    propagate_level(O[i], geometry_grid, iterations=k, factor=p)

# 4. Merging
# Compute sampling function and merge into single LPV M
for x in range(n):
    for y in range(n):
        for z in range(n):
            # Determine best level index
            min_idx = l - 1
            for i in range(l):
                # Map world coordinates to level i coordinates
                level_x = x // (2**i)
                level_y = y // (2**i)
                level_z = z // (2**i)
                
                # Check bounds for level i
                if level_x < (n // (2**i)) and level_y < (n // (2**i)) and level_z < (n // (2**i)):
                    idx_val = I[i][level_x][level_y][level_z]
                    if idx_val < min_idx:
                        min_idx = idx_val
            
            # Sample from the chosen level
            sample_x = x // (2**min_idx)
            sample_y = y // (2**min_idx)
            sample_z = z // (2**min_idx)
            
            M[x][y][z] = O[min_idx][sample_x][sample_y][sample_z]

# 5. Rendering
# Render scene using M as the light propagation volume
render_scene_with_lpv(M)
```

## Operating characteristics
*   **Time Complexity:** The propagation speed increases by almost 60% compared to uniform LPV because the number of iterations is reduced from $k=16$ (uniform) or $k=32$ (theoretical) to $k=4$ [S1, unofficial]. This speed increase offsets the overhead of downsampling, merging, and index volume management [S1, unofficial].
*   **Memory Complexity:** Full octrees are highly inefficient for sparse data, but suitable here because larger nodes represent coarse data [S1, unofficial]. The method imposes additional GPU memory cost compared to a single uniform grid [S1, unofficial].
*   **Measured Performance:** Renders the Sponza scene at 9 fps, which is 8% faster than the original LPV technique on an NVIDIA GeForce GTX 480 [S1, unofficial].
*   **Strengths:** Faster propagation due to hierarchical resolution; allows injection of regular point light sources; easier GPU implementation than sparse pointer-based octrees [S1, unofficial].
*   **Limitations:** Memory inefficiency for sparse data; intensity variations between octree levels exist but have little visual impact; light propagates further in lower levels leading to slightly higher overall ambient lighting [S1, unofficial].
*   **When to use:** When real-time global illumination is needed with point lights and performance is critical, and the scene geometry is dense enough to justify the memory overhead of a full octree [S1, unofficial].
*   **When not to use:** For scenes with very sparse geometry where memory efficiency is paramount, or when higher visual fidelity in indirect lighting is required than what 4 iterations can provide [S1, unofficial].

## Background
*   **Origin:** Proposed by John David Olovsson and Michael Doggett in 2013 [S1, unofficial].
*   **History:** It is an extension of Light Propagation Volumes (LPV) [KD10] and builds on the concept of storing pre-computed indirect illumination using spherical harmonics introduced by Sloan et al. [SKS02] [S1, unofficial]. It was implemented using CUDA and OpenSceneGraph with osgCompute [S1, unofficial].
*   **Usage:** Used in NVIDIA's CLPV implementation as a reference for propagation parameters [S1, unofficial]. Tested on the Sponza dataset [S1, unofficial].

## Subtopics

### Cascaded Light Propagation Volumes (CLPV)
Uses a small preset of grids with the same resolution but different extents/densities. Prioritizes spatial proximity to the viewer. [CLPV](4333-cascaded-light-propagation-volumes.md)

### Sparse Octree Light Propagation
Uses complex sparse pointer-based octrees with complex memory access patterns. [Sparse Octree LPV](4334-sparse-octree-light-propagation.md)

## Sources
[S1] Octree Light Propagation Volumes, https://lup.lub.lu.se/record/4002700, fetched 2026-10-26 (unofficial)
