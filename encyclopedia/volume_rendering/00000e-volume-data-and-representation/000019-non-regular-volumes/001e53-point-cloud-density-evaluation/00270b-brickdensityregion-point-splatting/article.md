# BrickDensityRegion Point Splatting


## Overview
BrickDensityRegion Point Splatting is a method for converting sparse point cloud data into a dense volumetric density representation within a hierarchical "brick" structure. It solves the problem of efficiently populating voxel grids with density values derived from point attributes (specifically O2 concentration) by projecting points onto the grid using a distance-weighted kernel. It differs from sibling methods like [Chamfer Distance](2446-chamfer-distance.md) or [Neighbor Counting](2905-neighbor-counting.md) by explicitly generating a continuous density field suitable for volume rendering, rather than computing discrete metrics or occupancy counts.

## How it works
The method maps individual points to a 3D voxel grid (brick) using normalized coordinates and a configurable splat radius.

**Inputs:**
- Point cloud data containing location ($P$) and O2 concentration ($O_2$).
- Volume resolution ($V_{res} = (X, Y, Z)$).
- Minimum location ($minLoc$) and location difference ($locDiff$) for normalization.
- Minimum O2 concentration ($minO_2$) and O2 difference ($O_2diff$) for normalization.
- Density multiplier ($m\_density\_mult$).

**Outputs:**
- A 3D voxel grid (brick) where each voxel stores a density value.

**Data Structures:**
- **Voxel Grid:** A 3D array of floats indexed by $(i, j, k)$.
- **Point:** Contains `getLocation()` (Vec3) and `getO2Concenration()` (float).

**Algorithm Steps:**

1.  **Normalize Point Location:**
    Calculate the normalized location $tmpLoc$ for a point $P$:
    $$tmpLoc = \frac{P - minLoc}{locDiff}$$
    where $tmpLoc.x = \frac{P.x - minLoc.x}{locDiff.x}$, etc. [S1, unofficial]

2.  **Normalize O2 Concentration:**
    Calculate the normalized O2 value $tmpO_2$:
    $$tmpO_2 = \frac{O_2 - minO_2}{O_2diff}$$
    [S1, unofficial]

3.  **Map to Voxel Coordinates:**
    Determine the integer voxel coordinates $(end\_x, end\_y, end\_z)$:
    $$end\_x = \lfloor tmpLoc.x \cdot X \rfloor$$
    $$end\_y = \lfloor tmpLoc.y \cdot Y \rfloor$$
    $$end\_z = \lfloor tmpLoc.z \cdot Z \rfloor$$
    [S1, unofficial]

4.  **Calculate Base Density:**
    $$end\_density = tmpO_2 \cdot m\_density\_mult$$
    [S1, unofficial]

5.  **Determine Splat Radius (splatVal):**
    The splat radius is an integer determined by thresholds on $tmpO_2$:
    - If $tmpO_2 < 0.4$: $splatVal = 0$
    - If $0.4 \le tmpO_2 < 0.6$: $splatVal = 1$
    - If $0.6 \le tmpO_2 < 0.9$: $splatVal = 2$
    - If $0.9 \le tmpO_2 < 0.95$: $splatVal = 3$
    - Otherwise ($tmpO_2 \ge 0.95$): $splatVal = 4$
    [S1, unofficial]

6.  **Boundary Check:**
    Only proceed if the voxel is strictly inside the grid boundaries:
    $$1 < end\_x < X-2 \quad \text{AND} \quad 1 < end\_y < Y-2 \quad \text{AND} \quad 1 < end\_z < Z-2$$
    If any coordinate is $\le 1$ or $\ge X-2$ (etc.), the point is ignored. [S1, unofficial]

7.  **Splatting (Weighted Update):**
    For valid points, iterate over the cubic neighborhood defined by the splat radius and update voxels.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splat_point_to_brick(point, min_loc, loc_diff, min_o2, o2_diff, 
                         vol_res, density_mult, brick_size):
    """
    point: object with getLocation() -> Vec3, getO2Concenration() -> float
    vol_res: Vec3 (X, Y, Z)
    brick_size: Vec3 (size_x, size_y, size_z)
    """
    # 1. Normalize location
    tmp_loc_x = (point.getLocation().x - min_loc.x) / loc_diff.x
    tmp_loc_y = (point.getLocation().y - min_loc.y) / loc_diff.y
    tmp_loc_z = (point.getLocation().z - min_loc.z) / loc_diff.z
    
    # 2. Normalize O2
    tmp_o2 = (point.getO2Concenration() - min_o2) / o2_diff
    
    # 3. Map to integer voxel coordinates
    end_x = int(tmp_loc_x * vol_res.x)
    end_y = int(tmp_loc_y * vol_res.y)
    end_z = int(tmp_loc_z * vol_res.z)
    
    # 4. Calculate base density
    base_density = tmp_o2 * density_mult
    
    # 5. Determine splat radius
    if tmp_o2 < 0.4:
        splat = 0
    elif tmp_o2 < 0.6:
        splat = 1
    elif tmp_o2 < 0.9:
        splat = 2
    elif tmp_o2 < 0.95:
        splat = 3
    else:
        splat = 4
        
    # 6. Boundary check: strictly between 1 and resolution-2
    if not (1 < end_x < vol_res.x - 2 and 
            1 < end_y < vol_res.y - 2 and 
            1 < end_z < vol_res.z - 2):
        return

    # 7. Splatting
    # Calculate max distance for kernel
    max_dist = (splat * splat * 3) ** 0.5
    
    # Iterate over cubic neighborhood clamped to brick boundaries
    start_x = max(0, end_x - splat)
    end_x_iter = min(brick_size.x - 1, end_x + splat)
    
    start_y = max(0, end_y - splat)
    end_y_iter = min(brick_size.y - 1, end_y + splat)
    
    start_z = max(0, end_z - splat)
    end_z_iter = min(brick_size.z - 1, end_z + splat)
    
    for i in range(start_x, end_x_iter + 1):
        for j in range(start_y, end_y_iter + 1):
            for k in range(start_z, end_z_iter + 1):
                # Euclidean distance from splat center
                dist = ((end_x - i)**2 + (end_y - j)**2 + (end_z - k)**2) ** 0.5
                
                # Contribution weight
                if splat == 0:
                    contrib = 1.0
                else:
                    contrib = 1.0 - (dist / max_dist)
                
                # Add weighted value to voxel
                # Note: The source uses 'val * contrib * contrib'
                # Assuming 'val' is base_density
                voxel_value = base_density * contrib * contrib
                
                # Add to existing voxel value (accumulation)
                brick[i, j, k] += voxel_value
```

## Subtopics

### BrickDensityRegion Splatting
This variant describes the specific kernel application logic used within the BrickDensityRegion framework. It defines the splatting radius calculation as $max = \sqrt{splat^2 \cdot 3}$ and uses a linear falloff kernel $contrib = 1 - (dist / max)$. If the splat radius is 0, the contribution is constant at 1. The weight is squared ($contrib^2$) before being multiplied by the base density and added to the voxel. This ensures a smooth transition of density values from the point center to the periphery of the splat. [S1, unofficial]

## Operating characteristics
- **Time Complexity:** $O(N \cdot R^3)$ where $N$ is the number of points and $R$ is the splat radius. Since $R$ is small (0-4), this is effectively linear with respect to point count.
- **Memory Complexity:** $O(X \cdot Y \cdot Z)$ for the voxel grid.
- **Strengths:** Efficiently converts sparse points to dense volumes; handles variable density via O2 concentration thresholds; simple linear kernel is fast to compute.
- **Limitations:** Boundary conditions are strict (points near edges are dropped); resolution is fixed by $vol\_res$; the linear kernel may introduce artifacts if splat radii vary significantly.
- **When to use:** When a quick, approximate density field is needed for visualization or further processing from a point cloud with associated scalar attributes.

## Background
The method originates from the `Mr-Voxel` project by cgibson, an unofficial implementation or tool related to voxel-based volume rendering and data loading. It is designed to load point cloud data into a brick-based volume structure for efficient rendering. [S1, unofficial]

## Sources
[S1] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
