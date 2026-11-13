# Octree Light Propagation Volumes

Also called: Light Propagation Volumes with Octrees

## Overview
Octree Light Propagation Volumes (OLPV) is a real-time global illumination technique that represents Light Propagation Volumes using a full octree data structure to allow light from regular point light sources to be injected, enabling faster propagation and rendering by performing propagation at lower resolutions for distant areas [S1, S3, unofficial]. It sets itself apart from its sibling [Cascaded Light Propagation Volumes](4333-cascaded-light-propagation-volumes.md) by using a single hierarchical volume covering the same world space with varying resolutions, prioritizing proximity to scene geometry and reflected indirect light rather than viewer proximity [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene geometry, light sources (omnidirectional point lights), Reflective Shadow Maps (RSMs).
*   **Outputs:** A merged uniform Light Propagation Volume (LPV) $M$ used for rendering indirect illumination.

**Data Structures:**
1.  **Octree Levels ($O_i$):** A full octree with $l = \log_2(n) + 1$ levels, where $n$ is the base resolution (a power of two). Level $i=0$ has size $n \times n \times n$. Level $i$ has size $2^{n-i} \times 2^{n-i} \times 2^{n-i}$? No, the text states indices range $[0, ..., 2^{n-i}]$? Let's re-read carefully.
    *   Source [S1, unofficial]: "indices $\hat{x}, \hat{y}, \hat{z} \in [0, ..., 2^{n-i}]$". Wait, if $n=32$, $2^{32}$ is huge. The text says "size $n \times n \times n$" for level 0. And "total levels $l = \log_2 n + 1$".
    *   Let's check the downsampling formula: $O_i(x,y,z)$ averages 8 elements from $O_{i-1}$. This implies $O_i$ is half the linear dimension of $O_{i-1}$.
    *   If $O_0$ is $n \times n \times n$, then $O_1$ is $(n/2) \times (n/2) \times (n/2)$.
    *   The index range in [S1, unofficial] says $\hat{x}, \hat{y}, \hat{z} \in [0, ..., 2^{n-i}]$. This looks like a typo in the source text or a specific notation where $n$ is the exponent?
    *   Re-reading [S1, unofficial]: "An octree is assigned a size, or resolution n... highest resolution level... size $n \times n \times n$... total levels $l = \log_2 n + 1$... indices $\hat{x}, \hat{y}, \hat{z} \in [0, ..., 2^{n-i}]$."
    *   If $n=32$, $\log_2 32 = 5$, so $l=6$. Levels $0..5$.
    *   If the index range is $2^{n-i}$, for $i=0$, range is $2^{32}$? That contradicts $n \times n \times n$.
    *   However, [S3, unofficial] says: "Index range for coordinates $\hat{x}, \hat{y}, \hat{z}$ on level $i$ is $[0, ..., 2^{n-i}]$."
    *   There is a contradiction or a specific definition of $n$ in the source. Let's look at the downsampling: $O_i(x,y,z)$ covers same volume as 8 elements of $O_{i-1}$. This implies linear dimension halves.
    *   If $O_0$ is $N \times N \times N$, $O_1$ is $N/2 \times N/2 \times N/2$.
    *   The source likely meant the dimension is $2^{k-i}$ where $k$ is related to $n$. Or perhaps $n$ in the index formula refers to the *log* of the resolution?
    *   Let's look at the parameter: "Octree Resolution n... power of two... size of highest resolution level... $n \times n \times n$."
    *   If $n=32$, dimension is 32.
    *   The formula $2^{n-i}$ is almost certainly a typo in the source for $2^{\log_2 n - i}$ or $n/2^i$. Given the context of "full octree" and standard downsampling, the dimension of level $i$ is $n / 2^i$.
    *   I will use the dimension $D_i = n / 2^i$ for clarity, as the averaging formula $O_i(x,y,z) = \frac{1}{8} \sum O_{i-1}(2x, 2y, 2z)$ confirms the halving.
    *   Memory Layout: Linear memory allocating all nodes down to a pre-defined level. Each level $i$ is a 3D grid.

2.  **Index Volume ($I_i$):** A hierarchy of structures $I_i$ for $i=0, ..., l-1$.
    *   Layout: Same 3D grid layout as the data storage.
    *   Size: $I_0$ has size $n \times n \times n$.
    *   Content: Integer indices $0, ..., l-1$.
    *   Initialization: All elements $I_i(x,y,z)$ are initialized to $l-1$ [S1, S3, unofficial].
    *   Purpose: Stores the octree level to sample for a given world space position. Lower values indicate proximity to geometry/reflected light [S3, unofficial].

**Numbered Steps:**

1.  **Light Injection:** Render the scene from each light source into six Reflective Shadow Maps (RSMs) [S1, S3, unofficial]. Convert the texels in the RSMs into Virtual Point Lights (VPLs) [S1, S3, unofficial]. Inject the contributions of the VPLs into the octree level $O_0$ using the same method as the original LPV technique (two-band spherical harmonics) [S1, S3, unofficial].

2.  **Downsampling:** For each level $i$ from $1$ to $l-1$, populate element $O_i(x,y,z)$ by averaging the corresponding eight elements from the previous level $O_{i-1}$ [S1, S3, unofficial]. This step is performed after and without interaction with the injection step [S1, S3, unofficial].
    $$O_i(x, y, z) = \frac{1}{8} \sum_{\hat{x}=2x}^{2x+1} \sum_{\hat{y}=2y}^{2y+1} \sum_{\hat{z}=2z}^{2z+1} O_{i-1}(\hat{x}, \hat{y}, \hat{z})$$
    The indices $\hat{x}, \hat{y}, \hat{z}$ range over the children of $(x,y,z)$ in the previous level [S1, S3, unofficial].

3.  **Index Volume Population:** The index volume $I$ is populated to reflect the octree structure. Generally, lower values are assigned close to geometry and higher values in empty areas [S3, unofficial]. The specific algorithm for populating $I$ is not detailed in the provided notes, but it determines the sampling level [S1, unofficial].

4.  **Propagation:** Perform light propagation individually on each level of the octree using an iterative process where each element receives contributions from surrounding elements [S1, S3, unofficial]. Use the light propagation scheme from Kaplanyan and Dachsbacher [KD10], which includes maintaining a separate grid containing the geometry of the scene [S1, S3, unofficial]. Light on lower levels (coarser resolution) propagates further each iteration, achieving coverage of the entire scene with fewer iterations [S1, S3, unofficial].
    *   Number of iterations $k=4$ [S1, S3, unofficial].
    *   Propagation factor $p=3$ [S1, S3, unofficial].

5.  **Merging:** After propagation, sample the octree using the scheme from Equation 1 to get values $S(x,y,z)$, then merge them into a single traditional uniform LPV $M$ [S1, S3, unofficial].
    $$M(x, y, z) = S(x, y, z)$$
    This step is necessary because directly sampling all levels of the octree and index volume during rendering is impractical [S1, S3, unofficial].

6.  **Rendering:** Render the scene using the merged single LPV $M$ [S1, S3, unofficial]. Use the exact same rendering step as in the traditional LPV technique [S1, S3, unofficial]. The volume is sampled only once in each $4 \times 4$ region to allow for high quality shadows while maintaining reasonable performance [S1, S3, unofficial].

**Sampling Function:**
When sampling the octree (for merging or debugging), the world position is mapped to an element in the index volume. The value in the index volume is used to do a second mapping into the correct octree level. The complication is that there are $l$ index volume elements that could be picked for each world space position. To solve this, the element containing the minimum value is used [S1, S3, unofficial].

$$I(x, y, z) = \min_i \left( I_i \left( \left\lfloor \frac{x}{2^i} \right\rfloor, \left\lfloor \frac{y}{2^i} \right\rfloor, \left\lfloor \frac{z}{2^i} \right\rfloor \right) \right)$$

The resulting sampling in the octree corresponds to:
$$O_{I(x,y,z)} \left( \left\lfloor \frac{x}{2^{I(x,y,z)}} \right\rfloor, \left\lfloor \frac{y}{2^{I(x,y,z)}} \right\rfloor, \left\lfloor \frac{z}{2^{I(x,y,z)}} \right\rfloor \right)$$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def octree_light_propagation_volumes(scene, lights, n, k=4, p=3):
    """
    n: Base resolution (power of two), e.g., 32.
    k: Number of propagation iterations.
    p: Propagation factor.
    """
    l = int(math.log2(n)) + 1
    
    # 1. Initialize Octree Levels and Index Volume
    # O[i] is a 3D grid of size (n // 2**i) x (n // 2**i) x (n // 2**i)
    O = [create_3d_grid(n // (2**i)) for i in range(l)]
    # I[i] is a 3D grid of size n x n x n for all i
    I = [create_3d_grid(n) for _ in range(l)]
    
    # Initialize Index Volume to max level
    for i in range(l):
        I[i].fill(l - 1)
        
    # 2. Light Injection
    # Render RSMs and create VPLs
    vpls = create_vpls_from_rsm(scene, lights)
    
    # Inject VPLs into O[0] using standard LPV injection (2-band SH)
    inject_vpls_into_lp(O[0], vpls)
    
    # 3. Downsampling
    for i in range(1, l):
        prev_level = O[i-1]
        curr_level = O[i]
        # Dimensions of curr_level are half of prev_level
        dim_curr = curr_level.shape[0]
        
        for x in range(dim_curr):
            for y in range(dim_curr):
                for z in range(dim_curr):
                    # Average 8 children from previous level
                    sum_val = 0.0
                    for dx in range(2):
                        for dy in range(2):
                            for dz in range(2):
                                sum_val += prev_level[2*x + dx, 2*y + dy, 2*z + dz]
                    curr_level[x, y, z] = sum_val / 8.0

    # 4. Populate Index Volume (I)
    # Note: Specific algorithm for populating I based on geometry proximity
    # is not detailed in sources, but generally assigns lower indices near geometry.
    # [G] Placeholder for geometry-based index population logic.
    populate_index_volume(I, scene, O, l)

    # 5. Propagation
    for i in range(l):
        propagate_level(O[i], I, k, p)

    # 6. Merging
    # Create merged LPV M of size n x n x n
    M = create_3d_grid(n)
    
    for x in range(n):
        for y in range(n):
            for z in range(n):
                # Determine sampling level
                min_idx = l - 1
                for i in range(l):
                    # Map world coord (x,y,z) to index volume coord
                    # The source formula uses floor(x/2^i)
                    idx_i = I[i][x // (2**i), y // (2**i), z // (2**i)]
                    if idx_i < min_idx:
                        min_idx = idx_i
                
                # Sample from the determined octree level
                level = min_idx
                # Map world coord to octree level coord
                ox = x // (2**level)
                oy = y // (2**level)
                oz = z // (2**level)
                
                M[x, y, z] = O[level][ox, oy, oz]

    return M

def propagate_level(level_grid, index_volume, k, p):
    """
    Propagate light in a single level grid using LPV scheme.
    """
    for iteration in range(k):
        new_grid = copy(level_grid)
        # Standard LPV propagation: each element receives contribution from neighbors
        # [G] Implementation of Kaplanyan and Dachsbacher [KD10] propagation scheme
        # involving geometry occlusion and spherical harmonic rotation/scaling.
        propagate_lp_v(level_grid, new_grid, p)
        level_grid = new_grid
```

## Operating characteristics
*   **Time Complexity:** Propagation is performed on each level individually. Light on lower levels (coarser resolution) propagates further, reducing the total number of iterations required from $k=16$ (uniform LPV) or $k=32$ (theoretical) to $k=4$ [S1, S3, unofficial]. This increases propagation speed by almost 60% [S1, S3, unofficial].
*   **Memory Complexity:** Uses a full octree, which is highly inefficient for sparse data but suitable here because larger nodes represent coarse data [S1, S3, unofficial]. The index volume adds overhead [S1, S3, unofficial].
*   **Measured Performance:** Renders the Sponza scene at 9 fps, which is 8% faster than the original LPV technique on an NVIDIA GeForce GTX 480 [S1, S3, unofficial]. NVIDIA's CLPV implementation reaches 60-80 fps with a single directed light source, whereas this implementation uses 18 RSMs [S3, unofficial].
*   **Strengths:** Faster propagation due to fewer iterations; allows injection of regular point light sources; simpler memory access patterns than sparse pointer-based octrees [S1, S3, unofficial].
*   **Limitations:** Full octrees are memory inefficient for sparse data; small intensity variations between levels exist but have little visual impact [S1, S3, unofficial]; sampling all octree levels in the rendering pipeline is impractical, requiring a merge step [S1, S3, unofficial].
*   **When to use:** When proximity to scene geometry and reflected indirect light is prioritized over viewer proximity, and point light injection is desired [S1, S3, unofficial].
*   **When not to use:** When memory is extremely constrained and the scene is very sparse, or when viewer-centric cascading is preferred [S1, S3, unofficial].

## Background
*   **Origin:** Proposed by John David Olovsson and Michael Doggett in 2013 [S1, S3, unofficial].
*   **History:** Builds on the original Light Propagation Volumes technique by Kaplanyan and Dachsbacher [KD10] and the concept of storing pre-computed indirect illumination using spherical harmonics by Sloan et al. [SKS02]. It contrasts with Crassin et al.'s sparse pointer-based octree technique [S1, S3, unofficial].
*   **Usage:** Implemented using OpenSceneGraph with osgCompute and CUDA [S1, S3, unofficial]. Tested on the Sponza dataset [S1, S3, unofficial]. Used in NVIDIA's CLPV implementation as a reference for propagation parameters [S1, S3, unofficial].

## Sources
[S1] Octree Light Propagation Volumes, https://lup.lub.lu.se/record/4002700, fetched 2026-10-26 (unofficial)
[S2] Octree Light Propagation Volumes, https://core.ac.uk/download/83664325.pdf, fetched 2026-10-30 (unofficial)
[S3] Octree Light Propagation Volumes, https://core.ac.uk/works/100893913, fetched 2026-11-03 (unofficial)
