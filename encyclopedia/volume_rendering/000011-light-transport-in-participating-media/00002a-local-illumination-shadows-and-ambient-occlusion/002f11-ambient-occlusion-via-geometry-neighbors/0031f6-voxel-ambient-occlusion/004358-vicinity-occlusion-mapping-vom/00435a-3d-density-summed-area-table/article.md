# 3D Density Summed Area Table

Also called: 3D SAT Occlusion, 3D SAT for AO, Density SAT, Density Summed Area Table, Volumetric Ambient Occlusion (DSAT)

## Overview
The 3D Density Summed Area Table (DSAT) is a view-independent ambient occlusion technique for volume rendering that simulates occlusion at the voxel level by evaluating the local density environment [S2, unofficial]. It supports semi-transparent structures and is more accurate than 2D image-space approaches, though it incurs higher preprocessing costs and memory usage [S1]. It sets itself apart from opacity-based SATs by storing raw density values, allowing the occlusion factor to update dynamically when the transfer function changes without recomputing the table [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset $V$ of density values with dimensions $W \times H \times D$.
*   **Output:** A 3D Density Summed Area Table (DSAT) of the same dimensions, and a rendered volume image with ambient occlusion applied to opaque samples.

**Data Structures:**
*   **Volume $V$:** A 3D array where $V[x, y, z]$ represents the density at voxel coordinates $(x, y, z)$.
*   **DSAT:** A 3D array of the same size as $V$. Each cell $DSAT[x, y, z]$ stores the cumulative sum of density values from the origin $(0, 0, 0)$ to $(x, y, z)$.

**Formulas:**
The DSAT is defined recursively to allow efficient region queries:
$$ DSAT[x, y, z] = \sum_{i=0}^{x} \sum_{j=0}^{y} \sum_{k=0}^{z} V[i, j, k] $$

To compute the sum of density values for an arbitrary cuboid region defined by corners $(x_1, y_1, z_1)$ and $(x_2, y_2, z_2)$, eight queries on the DSAT are required:
$$ Sum = DSAT[x_2, y_2, z_2] - DSAT[x_1-1, y_2, z_2] - DSAT[x_2, y_1-1, z_2] - DSAT[x_2, y_2, z_1-1] + DSAT[x_1-1, y_1-1, z_2] + DSAT[x_1-1, y_2, z_1-1] + DSAT[x_2, y_1-1, z_1-1] - DSAT[x_1-1, y_1-1, z_1-1] $$
*(Note: Terms with negative indices are treated as 0.)*

The average density $\bar{d}$ of the vicinity region is calculated as:
$$ \bar{d} = \frac{Sum}{N} $$
where $N$ is the number of voxels in the cuboid region.

The occlusion factor $O$ is derived by applying the current transfer function's opacity mapping $T$ to this average density:
$$ O = T(\bar{d}) $$

**Algorithm Steps:**

1.  **Preprocessing (CPU):**
    1.  Load the volume density data $V$.
    2.  Construct the 3D DSAT incrementally. For each voxel $(x, y, z)$, compute the sum based on previously computed DSAT values. The incremental construction follows the standard 3D prefix sum logic, similar to the 2D case but extended to three dimensions [S2, unofficial].
    3.  Store the resulting DSAT in memory.

2.  **Rendering (GPU):**
    1.  Pass the DSAT and the volume model $V$ to the GPU [S2, unofficial].
    2.  Perform ray casting through the volume.
    3.  For each sample point along the ray that is classified as opaque (based on the transfer function):
        a.  Define a vicinity cuboid region around the sample point. The size of this region determines the scale of occlusion [S2, unofficial].
        b.  Query the DSAT using the eight-corner formula to obtain the sum of densities in this region.
        c.  Calculate the average density $\bar{d}$ by dividing the sum by the number of voxels in the region.
        d.  Map $\bar{d}$ to an opacity value using the current transfer function $T(\bar{d})$. This value serves as the ambient occlusion factor.
        e.  Modify the shading of the sample by reducing its lighting contribution according to the occlusion factor [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_3d_sat(volume):
    """
    Computes the 3D Density Summed Area Table from a volume.
    volume: 3D array of density values V[x, y, z]
    Returns: 3D array DSAT[x, y, z]
    """
    W, H, D = volume.shape
    dsat = np.zeros((W, H, D), dtype=np.float32)
    
    # Incremental construction
    for x in range(W):
        for y in range(H):
            for z in range(D):
                # Sum of current voxel
                current = volume[x, y, z]
                
                # Add contributions from previous indices in each dimension
                # Using boundary checks for indices < 0
                from_x = dsat[x-1, y, z] if x > 0 else 0
                from_y = dsat[x, y-1, z] if y > 0 else 0
                from_z = dsat[x, y, z-1] if z > 0 else 0
                
                # Subtract overlaps added twice
                from_xy = dsat[x-1, y-1, z] if x > 0 and y > 0 else 0
                from_xz = dsat[x-1, y, z-1] if x > 0 and z > 0 else 0
                from_yz = dsat[x, y-1, z-1] if y > 0 and z > 0 else 0
                
                # Add back the triple overlap subtracted three times
                from_xyz = dsat[x-1, y-1, z-1] if x > 0 and y > 0 and z > 0 else 0
                
                dsat[x, y, z] = current + from_x + from_y + from_z - from_xy - from_xz - from_yz + from_xyz
                
    return dsat

def query_region_sum(dsat, x1, y1, z1, x2, y2, z2):
    """
    Computes the sum of values in the cuboid defined by (x1,y1,z1) to (x2,y2,z2)
    using the 3D DSAT.
    """
    def get_val(x, y, z):
        if x < 0 or y < 0 or z < 0:
            return 0
        # Assuming dsat is within bounds for valid queries
        return dsat[x, y, z]

    return (get_val(x2, y2, z2) 
            - get_val(x1-1, y2, z2) 
            - get_val(x2, y1-1, z2) 
            - get_val(x2, y2, z1-1) 
            + get_val(x1-1, y1-1, z2) 
            + get_val(x1-1, y2, z1-1) 
            + get_val(x2, y1-1, z1-1) 
            - get_val(x1-1, y1-1, z1-1))

def apply_ao_sample(ray_sample, dsat, vicinity_size, transfer_function):
    """
    Calculates ambient occlusion for a single sample point.
    """
    # Define vicinity region boundaries
    # Assuming ray_sample has coordinates (x, y, z)
    x, y, z = ray_sample.coords
    half_size = vicinity_size // 2
    
    x1, y1, z1 = max(0, x - half_size), max(0, y - half_size), max(0, z - half_size)
    x2, y2, z2 = min(dsat.shape[0]-1, x + half_size), min(dsat.shape[1]-1, y + half_size), min(dsat.shape[2]-1, z + half_size)
    
    # Get sum of densities in vicinity
    density_sum = query_region_sum(dsat, x1, y1, z1, x2, y2, z2)
    
    # Calculate number of voxels in region
    num_voxels = (x2 - x1 + 1) * (y2 - y1 + 1) * (z2 - z1 + 1)
    
    # Average density
    avg_density = density_sum / num_voxels
    
    # Occlusion factor from transfer function
    occlusion_factor = transfer_function.opacity(avg_density)
    
    return occlusion_factor
```

**Parameters and Edge Cases:**
*   **Vicinity Size:** The size of the cuboid region queried for average density affects the smoothness and scale of the occlusion. Larger regions produce smoother, broader occlusion but may blur fine details [S2, unofficial].
*   **Density Variability:** If density values vary significantly within the vicinity region, the average density may not accurately represent the local opacity, leading to artifacts [S2, unofficial].
*   **Transfer Function Changes:** Since DSAT stores density, not opacity, changing the transfer function does not require recomputing the DSAT. Only the final occlusion factor calculation changes [S2, unofficial].
*   **View Independence:** The DSAT is view-independent, so it does not need to be recomputed when the camera view changes [S1].

## Operating characteristics
*   **Time Complexity:** Preprocessing is $O(W \times H \times D)$ for constructing the DSAT. Per-sample rendering cost is constant $O(1)$ for the DSAT queries, but requires 8 texture accesses per sample for the region sum [S1].
*   **Memory Complexity:** Requires storing the DSAT, which has the same size as the volume data, effectively doubling the memory footprint for the density data [S1].
*   **Performance:** Higher preprocessing cost on the CPU compared to screen-space methods. However, it avoids GPU-CPU transfers during rendering, leading to higher frame rates in interactive scenarios compared to dynamic screen-space approaches [S2, unofficial].
*   **Accuracy:** More accurate than 2D image-space approaches because it considers the full 3D neighborhood, including semi-transparent structures [S1].
*   **Limitations:** Artifacts can occur if the density variability in the vicinity is high, as the average density may not map correctly to the actual local opacity [S2, unofficial].

## Background
The 3D Density Summed Area Table was introduced as a method to enhance volume visualization by providing view-independent ambient occlusion. It builds upon the concept of Summed Area Tables used in 2D image processing and extends it to 3D volumes. The method was described in the context of enhancing perception in volume visualization, specifically addressing the limitations of depth-based screen-space occlusion methods that fail to account for semi-transparent structures [S2, unofficial]. It is related to other SAT-based techniques such as those by Desgranges and Engel [S1].

## Sources
[S1] Interactive Volume Rendering with Volumetric Illumination, https://doi.org/10.2312/conf/eg2012/stars/053-074, fetched 2026-10-26
[S2] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
