# Per-Region Gradient Averaging

Also called: Region-Averaged Gradient, Binary Neighbor Gradient Averaging

## Overview
Per-Region Gradient Averaging computes the gradient magnitude for each voxel in a scalar volume by aggregating directional gradients across multiple defined scalar regions. It distinguishes itself from standard finite difference methods by using a binary neighbor check to detect region boundaries, assigning high weights to neighbors within the same region to enforce spatial coherence. This method is primarily used to smooth learned volume representations and prevent high-frequency noise, serving as a preprocessing step for regularization techniques like Total Variation Gradient Addition.

## How it works
**Inputs and Outputs:**
- Input: A 3D volume grid of scalar values (representing multiple regions) and a list of defined regions.
- Output: A 3D grid of gradient magnitudes, one per voxel.

**Data Structures:**
- `self.data`: A flat list or array of `VoxelGradient` objects, with size $dim_x \times dim_y \times dim_z$.
- `VoxelGradient`: A structure containing:
  - `prev_gx`, `prev_gy`, `prev_gz`: Floats storing the running average of directional gradients.
  - `magnitude`: Float storing the final computed gradient magnitude.

**Numbered Steps:**
1. Initialize the volume data structure `self.data` with `VoxelGradient` objects for each voxel.
2. Replace the original volume scalar data with a boolean mask indicating region membership for each voxel.
3. For each voxel $(x, y, z)$ in the grid:
   a. Initialize `prev_gx = 0`, `prev_gy = 0`, `prev_gz = 0`, and counter `n = 0`.
   b. Iterate through each region in the `regions` list:
      i. Check the 6 spatial neighbors (±x, ±y, ±z).
      ii. For each neighbor, determine if it belongs to the current region.
      iii. Assign a directional weight: $1000$ if the neighbor is in the region, $0$ otherwise.
      iv. Compute the directional gradient components ($g_x, g_y, g_z$) using the weighted neighbors.
      v. If the gradient is non-zero, update the running average:
          $$n_{new} = n + 1$$
          $$prev\_gx = \frac{prev\_gx \times n + g_x}{n_{new}}$$
          (Repeat for $y$ and $z$).
      vi. Update `n = n_new`.
   c. After processing all regions, set the voxel's gradient vector to $(prev\_gx, prev\_gy, prev\_gz)$.
   d. Compute the magnitude: $\sqrt{prev\_gx^2 + prev\_gy^2 + prev\_gz^2}$.
4. Store the magnitude in the `VoxelGradient` object.
5. Implement lazy evaluation for `get_max_gradient_magnitude`: On the first call, scan `self.data` to find the maximum magnitude.

**Formulas:**
The directional gradient in the x-direction is computed as:
$$g_x = \frac{x_{plus} - x_{minus}}{2}$$
Where:
- $x_{plus} = 1000$ if $V(x+1, y, z) \in \text{region}$, else $0$.
- $x_{minus} = 1000$ if $V(x-1, y, z) \in \text{region}$, else $0$.
- Similar definitions apply for $g_y$ and $g_z$ using $y_{\pm}$ and $z_{\pm}$.

**Parameters:**
- Weight $W = 1000$: A binary weight assigned to neighbors belonging to the current region to amplify boundary detection. [S1, unofficial]

**Edge Cases and Pitfalls:**
- Boundary voxels access neighbors outside the volume bounds. This is handled by a helper function `get_volume_voxel` which is assumed to clamp coordinates or return $0$ for out-of-bounds access. [S1, unofficial]
- Precision issues may arise from the large weight value ($1000$) if not handled carefully, though the averaging process mitigates extreme spikes.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_per_region_gradients(volume_data, regions, dim_x, dim_y, dim_z):
    # Initialize data structure
    voxels = []
    for _ in range(dim_x * dim_y * dim_z):
        voxels.append(VoxelGradient())

    # Replace volume data with boolean mask for regions
    # Assuming volume_data is a 3D array of scalar values
    region_mask = get_region_mask(volume_data, regions)

    for x in range(dim_x):
        for y in range(dim_y):
            for z in range(dim_z):
                prev_gx = 0.0
                prev_gy = 0.0
                prev_gz = 0.0
                n = 0

                for region in regions:
                    gx = 0.0
                    gy = 0.0
                    gz = 0.0

                    # Check 6 neighbors
                    # x direction
                    x_plus_in = is_in_region(region_mask, x + 1, y, z, region)
                    x_minus_in = is_in_region(region_mask, x - 1, y, z, region)
                    
                    # y direction
                    y_plus_in = is_in_region(region_mask, x, y + 1, z, region)
                    y_minus_in = is_in_region(region_mask, x, y - 1, z, region)
                    
                    # z direction
                    z_plus_in = is_in_region(region_mask, x, y, z + 1, region)
                    z_minus_in = is_in_region(region_mask, x, y, z - 1, region)

                    # Compute weighted gradients
                    # Weight is 1000 if in region, 0 otherwise
                    w_plus_x = 1000.0 if x_plus_in else 0.0
                    w_minus_x = 1000.0 if x_minus_in else 0.0
                    gx = (w_plus_x - w_minus_x) / 2.0

                    w_plus_y = 1000.0 if y_plus_in else 0.0
                    w_minus_y = 1000.0 if y_minus_in else 0.0
                    gy = (w_plus_y - w_minus_y) / 2.0

                    w_plus_z = 1000.0 if z_plus_in else 0.0
                    w_minus_z = 1000.0 if z_minus_in else 0.0
                    gz = (w_plus_z - w_minus_z) / 2.0

                    # Update running average if gradient is non-zero
                    if gx != 0 or gy != 0 or gz != 0:
                        n += 1
                        prev_gx = (prev_gx * (n - 1) + gx) / n
                        prev_gy = (prev_gy * (n - 1) + gy) / n
                        prev_gz = (prev_gz * (n - 1) + gz) / n

                # Compute magnitude
                magnitude = (prev_gx**2 + prev_gy**2 + prev_gz**2)**0.5
                
                # Store result
                voxels[x * dim_y * dim_z + y * dim_z + z].magnitude = magnitude

    return voxels

def is_in_region(mask, x, y, z, region):
    # Helper to handle bounds and check region membership
    if not (0 <= x < dim_x and 0 <= y < dim_y and 0 <= z < dim_z):
        return False
    return mask[x][y][z] == region
```

## Subtopics

### [Total Variation Gradient Addition](2877-total-variation-gradient-addition.md)
Applies TV regularization to specific scalar fields using the aggregated gradient magnitudes from this method to enforce spatial coherence.

## Operating characteristics
- **Time Complexity:** $O(V \cdot R)$, where $V$ is the number of voxels and $R$ is the number of regions. Each voxel iterates through all regions and checks 6 neighbors.
- **Memory Complexity:** $O(V)$ for storing the gradient magnitudes and the region mask.
- **Strengths:** Effective at smoothing noise in learned volume representations by enforcing coherence within regions. Simple to implement.
- **Limitations:** Computationally expensive for volumes with many regions. The binary weight approach may oversimplify gradient information at complex boundaries.
- **When to use:** When using learned volume representations (e.g., NeRFs) that require regularization to prevent artifacts.
- **When not to use:** For standard physical volumes where accurate directional gradients are required for lighting or surface reconstruction.

## Background
Originated in the context of raycasting rendering implementations for learned volume representations, specifically in the `faviasono/raycastingrendering` repository. It is used as a preprocessing step to compute gradient magnitudes for regularization purposes.

## Sources
[S1] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
