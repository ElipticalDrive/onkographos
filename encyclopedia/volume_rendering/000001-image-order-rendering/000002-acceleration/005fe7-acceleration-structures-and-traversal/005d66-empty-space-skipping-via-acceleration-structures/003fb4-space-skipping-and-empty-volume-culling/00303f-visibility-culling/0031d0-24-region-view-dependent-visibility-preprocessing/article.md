# 24-Region View-Dependent Visibility Preprocessing


## Overview
A preprocessing technique that decomposes the domain of viewing directions into 24 regions on a directional cube to extract a lower number of potentially visible voxels compared to the standard 6-region decomposition, improving extraction rate and rendering speed [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A 3D volume dataset $V$ of size $N_x \times N_y \times N_z$, where each voxel has a binary boundary indicator $b(i, j, k)$ (1 if boundary, 0 otherwise) [S1, unofficial].
*   Output: A set of precomputed visibility masks $M_r$ for each of the 24 regions $r$, where $M_r(i, j, k) = 1$ if voxel $(i, j, k)$ is potentially visible from direction $r$, and 0 otherwise [S1, unofficial].

**Data Structures:**
*   $b(i, j, k)$: Binary grid storing boundary indicators [S1, unofficial].
*   $v(i, j, k)$: Binary visibility grid computed for a specific viewing direction region [S1, unofficial].
*   $M_r$: Array of visibility grids, one for each of the 24 regions [S1, unofficial].

**Steps:**
1.  **Decompose Viewing Directions:** Divide the viewing direction domain into 24 regions based on the signs of the $x$ and $y$ components, assuming the principal component of the viewing direction is the $z$ coordinate [S1, unofficial].
2.  **Compute Visibility for Each Region:** For each of the 24 regions, compute the visibility function $v(i, j, k)$ using the boundary grid [S1, unofficial].
3.  **Select Region at Render Time:** During rendering, determine the current viewing direction, identify which of the 24 regions it falls into, and select the corresponding precomputed visibility mask $M_r$ [S1, unofficial].

**Formula:**
For the case where the $x$ and $y$ components of the viewing direction are both positive, the visibility function $v(i, j, k)$ is defined as [S1, unofficial]:

$$
v(i, j, k) = \begin{cases} 
1 & \text{if } k = 0 \\
1 & \text{if } \exists l, m \text{ s.t. } i - 1 \leq l \leq i, \ j - 1 \leq m \leq j, \ v(l, m, k - 1) = 1, \ b(l, m, k - 1) = 0 \\
0 & \text{otherwise}
\end{cases}
$$

*   $k=0$: Voxels on the front-most slice (along the principal axis) are visible [S1, unofficial].
*   Recursive step: A voxel $(i, j, k)$ is visible if there exists a neighbor $(l, m)$ in the previous slice ($k-1$) such that the neighbor is visible ($v(l, m, k-1)=1$) and the neighbor is not a boundary voxel ($b(l, m, k-1)=0$) [S1, unofficial].
*   This condition checks only four voxels that could hide the current voxel from the specific viewing direction [S1, unofficial].
*   This condition is stronger than the general 6-region case (which checks 9 voxels), resulting in fewer extracted voxels [S1, unofficial].

**Parameters:**
*   Number of regions: 24 [S1, unofficial].
*   Principal axis: $z$ (assumed for the formula, but the method applies to any principal axis) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Preprocessing Time:** Decomposing into 24 regions increases preprocessing time compared to 6 regions [S1, unofficial].
*   **Storage:** Requires storing 24 visibility masks instead of 6, increasing memory usage [S1, unofficial].
*   **Direction Mapping:** The viewing direction must be accurately mapped to one of the 24 regions to select the correct precomputed data structure [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_visibility_quadrant(boundary):
    """
    Computes visibility mask for the viewing direction quadrant where x>0 and y>0.
    boundary: 3D list of shape (Nx, Ny, Nz) with 1 for boundary, 0 otherwise.
    Returns: 3D list of shape (Nx, Ny, Nz) with 1 for visible, 0 otherwise.
    """
    Nx, Ny, Nz = len(boundary[0][0]), len(boundary[0]), len(boundary)
    vis = [[[0] * Nz for _ in range(Ny)] for _ in range(Nx)]
    
    # Base case: k=0 is visible
    for i in range(Nx):
        for j in range(Ny):
            vis[i][j][0] = 1
            
    # Recursive step for k > 0
    for k in range(1, Nz):
        for i in range(Nx):
            for j in range(Ny):
                is_visible = False
                # Check the 4 neighbors in the previous slice: (i-1,j-1) to (i,j)
                for l in range(max(0, i - 1), min(Nx, i + 1)):
                    for m in range(max(0, j - 1), min(Ny, j + 1)):
                        if vis[l][m][k - 1] == 1 and boundary[l][m][k - 1] == 0:
                            is_visible = True
                            break
                    if is_visible:
                        break
                vis[i][j][k] = 1 if is_visible else 0
    return vis

def compute_all_24_regions(boundary):
    """
    Computes visibility masks for all 24 regions.
    The exact angular subdivision per sign quadrant is implementation-specific [G].
    This function demonstrates the core logic for one quadrant; others follow by flipping neighbor offsets.
    """
    masks = {}
    # Compute for the positive quadrant (x>0, y>0)
    masks['pos_pos'] = compute_visibility_quadrant(boundary)
    # Other 23 regions are computed similarly with sign-adapted neighbor ranges [G]
    return masks

def render_with_24_region_culling(volume_data, viewing_direction, visibility_masks):
    """
    Renders volume using precomputed 24-region visibility masks.
    """
    # Map viewing direction to one of the 24 regions based on sign combinations [G]
    region_id = get_region_id(viewing_direction)
    mask = visibility_masks[region_id]
    
    # Standard front-to-back compositing loop [G]
    for pixel in pixels:
        ray = generate_ray(pixel, viewing_direction)
        alpha_acc = 0.0
        color_acc = 0.0
        
        for t in ray:
            voxel = sample_volume(ray, t)
            if mask[voxel_coords] == 0:
                continue # Skip invisible voxel
            
            alpha_sample = voxel.opacity
            color_sample = voxel.color
            
            alpha_acc += alpha_sample * (1 - alpha_acc)
            color_acc += color_sample * alpha_sample * (1 - alpha_acc)
            
            if alpha_acc > 0.99:
                break # Early termination
        
        set_pixel_color(pixel, color_acc)
```

## Subtopics

### View-Dependent Preprocessing
A lightweight preprocessing step performed by each input processor (not rendering processor) before local rendering to optimize the subsequent image compositing step and select the appropriate octree level for adaptive rendering [S2]. It involves quantization (32-bit to 8-bit), central differencing for gradient vectors (lighting), and one-side differencing for temporal domain enhancement [S2]. The octree level is chosen based on image resolution, data resolution, and a user-specified limit on elements projected per pixel [S2]. This variant focuses on data preparation and level-of-detail selection rather than directional visibility culling [S2].

## Operating characteristics
*   **Time Complexity:** Preprocessing time increases with the number of regions, requiring more time than the 6-region method [S1, unofficial].
*   **Memory Complexity:** Requires storing 24 visibility masks instead of 6, increasing memory usage [S1, unofficial].
*   **Performance:** On a low resolution dataset, decomposing into 24 regions extracts 1.8% of voxels, compared to 3.4% for 6 regions [S1, unofficial]. On a high resolution volume, it extracts 0.9% compared to 1.8% for 6 regions [S1, unofficial]. This significant reduction in extracted voxels leads to faster rendering [S1, unofficial].
*   **Strengths:** Higher extraction rate efficiency (fewer voxels processed) compared to 6-region decomposition [S1, unofficial].
*   **Limitations:** Increased preprocessing time and storage requirements [S1, unofficial].

## Background
Proposed to improve the extraction rate of potentially visible voxels by refining the directional decomposition beyond the standard 6-region approach [S1, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S2] I/O Strategies for Parallel Rendering of Large Time-Varying Volume Data, https://doi.org/10.2312/egpgv/egpgv04/031-040, fetched 2026-10-26
