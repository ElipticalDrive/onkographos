# Ray-Box Intersection for Interaction

Also called: Mouse Interaction, Ray-Box Intersection for Voxel Editing, Raycasting for voxel selection, Smoke Injection

## Overview
This method determines the 3D location within a simulation volume where a user intends to inject smoke, velocity, or voxels based on 2D mouse input [S1, S2, unofficial]. It solves the problem of translating screen coordinates into a specific voxel index or world position for direct manipulation [S1, S2, unofficial]. Unlike [Voxel selection via pick ray intersection](893-voxel-selection-via-pick-ray-intersection.md), which computes a seed voxel for segmentation, this method focuses on the geometric intersection of a camera ray with the volume's bounding box to facilitate fluid injection or voxel placement [S1, S2, unofficial].

## How it works
**Inputs:**
- Mouse screen coordinates $(x_{screen}, y_{screen})$ in pixels.
- Window dimensions $WIDTH$ and $HEIGHT$.
- Camera parameters: position, view vectors, aspect ratio, and field of view (FOV).
- Simulation volume bounding box defined by $box_{min}$ and $box_{max}$.

**Outputs:**
- A boolean indicating if the ray intersects the volume.
- If intersected: the world position $p$ of the interaction point and the corresponding grid indices $(i_x, i_y, i_z)$.

**Data Structures:**
- Ray defined by origin $r_{origin}$ and direction $r_{dir}$ (normalized).
- Bounding box defined by minimum and maximum vectors.
- Grid resolution $N$ (assumed cubic for simplicity in source [S1, unofficial]).

**Steps:**

1.  **Convert Screen to NDC:**
    Transform mouse coordinates to Normalized Device Coordinates (NDC) $[-1, 1]$. The y-axis is flipped relative to standard OpenGL NDC if the screen y increases downwards [S1, unofficial]:
    $$
    x_{ndc} = \frac{2.0 \cdot x_{screen}}{WIDTH} - 1.0
    $$
    $$
    y_{ndc} = 1.0 - \frac{2.0 \cdot y_{screen}}{HEIGHT}
    $$

2.  **Construct Ray:**
    Generate the ray origin and direction from the camera. The direction is scaled by the aspect ratio and FOV to match the view frustum [S1, unofficial].

3.  **Ray-Box Intersection:**
    Intersect the ray with the simulation bounding box. The sources specify different bounding box extents depending on the context:
    - For fluid/smoke injection, the box is often $[-0.5, 0.5]$ [S1, unofficial].
    - For general voxel editing, the box may be defined as $[-1.0, 1.0]$ or derived from the grid [S2, unofficial].
    
    Compute $t_{near}$ and $t_{far}$, the entry and exit distances along the ray. If the ray does not intersect the box, the method returns false [S2, unofficial].

4.  **Calculate Interaction Point:**
    Compute the midpoint of the intersection segment to determine the world position $p$ [S1, unofficial]:
    $$
    t_{mid} = \frac{t_{near} + t_{far}}{2.0}
    $$
    $$
    p = r_{origin} + r_{dir} \cdot t_{mid}
    $$
    *Note:* In voxel editing contexts, if a previous hit was detected (hover), the hit position and normal may be used directly or adjusted [S2, unofficial].

5.  **Convert to Grid Indices:**
    Transform the world position $p$ to grid space. A common offset of $0.5$ is added to align with voxel centers, followed by scaling by the resolution $N$ [S1, unofficial]:
    $$
    p_{grid} = p + \begin{bmatrix} 0.5 \\ 0.5 \\ 0.5 \end{bmatrix}
    $$
    $$
    i_x = \lfloor p_{grid, x} \cdot N \rfloor
    $$
    $$
    i_y = \lfloor p_{grid, y} \cdot N \rfloor
    $$
    $$
    i_z = \lfloor p_{grid, z} \cdot N \rfloor
    $$
    The resulting indices $(i_x, i_y, i_z)$ identify the target voxel.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_box_interaction_for_editing(mouse_x, mouse_y, width, height, camera, box_min, box_max, grid_resolution):
    """
    Determines the voxel index for interaction based on mouse click.
    Returns (hit, voxel_index) or (False, None).
    """
    # 1. Convert to NDC
    x_ndc = (2.0 * mouse_x) / width - 1.0
    y_ndc = 1.0 - (2.0 * mouse_y) / height
    
    # 2. Construct Ray from Camera
    # camera.get_ray assumes it handles aspect ratio and FOV internally
    ray_origin, ray_dir = camera.get_ray(x_ndc, y_ndc)
    
    # 3. Ray-Box Intersection
    # Standard slab method for AABB intersection
    t_min = -float('inf')
    t_max = float('inf')
    
    # Calculate intersection intervals for each axis
    for i in range(3):
        if abs(ray_dir[i]) < 1e-8:
            # Ray is parallel to slab
            if ray_origin[i] < box_min[i] or ray_origin[i] > box_max[i]:
                return False, None
        else:
            inv_dir = 1.0 / ray_dir[i]
            t1 = (box_min[i] - ray_origin[i]) * inv_dir
            t2 = (box_max[i] - ray_origin[i]) * inv_dir
            
            # Ensure t1 < t2
            if t1 > t2:
                t1, t2 = t2, t1
            
            t_min = max(t_min, t1)
            t_max = min(t_max, t2)
            
            if t_min > t_max:
                return False, None
                
    if t_max < 0:
        return False, None
        
    # 4. Calculate Midpoint
    t_mid = (t_min + t_max) * 0.5
    p_world = ray_origin + ray_dir * t_mid
    
    # 5. Convert to Grid Indices
    # Add 0.5 offset for voxel center alignment
    p_grid = p_world + Vec3(0.5, 0.5, 0.5)
    
    ix = int(p_grid.x * grid_resolution)
    iy = int(p_grid.y * grid_resolution)
    iz = int(p_grid.z * grid_resolution)
    
    # Boundary check for grid indices
    if not (0 <= ix < grid_resolution and 0 <= iy < grid_resolution and 0 <= iz < grid_resolution):
        return False, None
        
    return True, (ix, iy, iz)
```

**Parameters:**
- $N$: Grid resolution. Typical values depend on simulation scale (e.g., 64, 128, 256) [S1, unofficial].
- Box Extents: $[-0.5, 0.5]$ for normalized simulation space [S1, unofficial]; $[-1.0, 1.0]$ or custom bounds for voxelizers [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Parallel Rays:** If the ray direction component is zero, the ray is parallel to the box face. If the origin is outside the slab, there is no intersection [S2, unofficial].
- **Hover State:** In voxel editing, if `hoverInfo.hasHit` is false after intersection, no interaction occurs, even if the ray hits the bounding box [S2, unofficial].
- **Placement Offset:** When adding a voxel adjacent to an existing one, the position is offset by the normal multiplied by a small epsilon (e.g., $0.01$) to avoid self-intersection [S2, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(1)$ for the ray-box intersection and coordinate transformation. The conversion to grid indices is also $O(1)$.
- **Memory Complexity:** $O(1)$ auxiliary memory.
- **Strengths:** Fast, deterministic, and simple to implement. Provides immediate feedback for user interaction.
- **Limitations:** The midpoint approximation assumes a uniform volume. For non-uniform grids or complex internal structures, stepping along the ray (as in [Voxel Raycasting for Block Interaction](3780-voxel-raycasting-for-block-interaction.md)) may be more accurate for finding the *first* solid voxel [S1, unofficial].
- **When to use:** Ideal for fluid injection, smoke placement, or simple voxel addition/removal where the exact surface intersection is less critical than the general region of interaction.

## Background
This method is a fundamental building block for interactive volume editing and fluid simulation interfaces. It bridges the gap between 2D user input (mouse) and 3D simulation state. The specific implementation details, such as the NDC conversion and midpoint calculation, are derived from practical implementations in fluid simulation repositories [S1, unofficial] and voxelization tools [S2, unofficial].

## Sources
[S1] tzhazuma/Fluid_simulation, https://github.com/tzhazuma/Fluid_simulation, fetched 2026-10-26 (unofficial)
[S2] GitHub - saurlax/model_voxelizer: A 3D model voxelization tool that ..., https://github.com/saurlax/model_voxelizer, fetched 2026-10-30 (unofficial)
