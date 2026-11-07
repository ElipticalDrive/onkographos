# Ray-Box Intersection for Interaction

Also called: Mouse Interaction, Ray-Box Intersection for Voxel Editing, Raycasting for voxel selection, Smoke Injection

## Overview
This method determines the 3D spatial coordinates corresponding to a 2D mouse click or hover event to enable direct manipulation of a voxel simulation volume, such as injecting smoke, velocity, or placing/removing voxels [S1, unofficial][S2, unofficial].

## How it works
**Inputs:** Mouse screen coordinates $(x_{screen}, y_{screen})$ in pixels, window dimensions $WIDTH$ and $HEIGHT$, camera state, simulation volume bounding box $[B_{min}, B_{max}]$ (specified as $[-0.5, 0.5]$ [S1, unofficial] or $[-1.0, 1.0]$ [S2, unofficial]), and grid resolution $N$.
**Outputs:** World position $P$, discrete grid indices $(i_x, i_y, i_z)$, and a boolean success flag.
**Data Structures:** Ray defined by origin $O$ and direction $D$; bounding box; hover info struct containing `hasHit` (bool), `position` (vec3), `normal` (vec3), and `node` (pointer/object) [S2, unofficial].

**Steps and Formulas:**

1. **Convert Screen Coordinates to NDC:**
   Map pixel coordinates to Normalized Device Coordinates (NDC) range $[-1, 1]$ with inverted y-axis [S1, unofficial]:
   $$ x_{ndc} = \frac{2.0 \cdot x_{screen}}{WIDTH} - 1.0 $$
   $$ y_{ndc} = 1.0 - \frac{2.0 \cdot y_{screen}}{HEIGHT} $$

2. **Construct Camera Ray:**
   Compute the ray origin $O$ and direction $D$ from the camera position and view vectors, scaled by aspect ratio and field of view [S1, unofficial][S2, unofficial].

3. **Intersect Ray with Simulation Box:**
   Initialize $t_{min} = -\infty$ and $t_{max} = +\infty$ [S2, unofficial]. For each axis $k \in \{x, y, z\}$, compute intersection distances with the box slabs:
   $$ t_1 = \frac{B_{min}[k] - O[k]}{D[k]}, \quad t_2 = \frac{B_{max}[k] - O[k]}{D[k]} $$
   $$ t_{min} = \max(t_{min}, \min(t_1, t_2)) $$
   $$ t_{max} = \min(t_{max}, \max(t_1, t_2)) $$
   If $t_{min} > t_{max}$, the ray does not intersect the grid [S2, unofficial].

4. **Calculate Intersection Point:**
   If the intersection is valid ($t_{min} \le t_{max}$), calculate the midpoint of the intersection segment [S1, unofficial]:
   $$ t_{mid} = \frac{t_{min} + t_{max}}{2.0} $$
   $$ P = O + D \cdot t_{mid} $$

5. **Convert to Grid Indices:**
   Transform the world position $P$ to discrete grid indices within the volume resolution $N \times N \times N$ [S1, unofficial]:
   $$ P_{grid} = P + \begin{bmatrix} 0.5 \\ 0.5 \\ 0.5 \end{bmatrix} $$
   $$ i_x = \lfloor P_{grid}[x] \cdot N \rfloor, \quad i_y = \lfloor P_{grid}[y] \cdot N \rfloor, \quad i_z = \lfloor P_{grid}[z] \cdot N \rfloor $$

6. **Handle Interaction Logic:**
   - **Removal:** If the action is voxel removal, return success only if `hoverInfo.hasHit` is true and `hoverInfo.node->isActive` is true [S2, unofficial].
   - **Addition:** If the action is voxel addition, offset the hit position by the surface normal $N_{surf}$ scaled by $\epsilon = 0.01$ to place the new voxel adjacent to the existing one: $P_{new} = P + N_{surf} \cdot 0.01$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_box_interaction(mouse_x, mouse_y, width, height, camera, box_min, box_max, grid_size):
    # 1. Convert to NDC
    x_ndc = (2.0 * mouse_x) / width - 1.0
    y_ndc = 1.0 - (2.0 * mouse_y) / height

    # 2. Construct Ray
    ray_origin, ray_dir = camera.get_ray(mouse_x, mouse_y)
    ray_dir = normalize(ray_dir)

    # 3. Ray-Box Intersection
    t_min = -float('inf')
    t_max = float('inf')

    for k in range(3):
        if abs(ray_dir[k]) < 1e-8:
            # Ray parallel to slab; fail if origin is outside bounds
            if ray_origin[k] < box_min[k] or ray_origin[k] > box_max[k]:
                return False, None, None
        else:
            t1 = (box_min[k] - ray_origin[k]) / ray_dir[k]
            t2 = (box_max[k] - ray_origin[k]) / ray_dir[k]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))

    if t_min > t_max:
        return False, None, None

    # 4. Calculate Midpoint
    t_mid = (t_min + t_max) * 0.5
    p_world = ray_origin + ray_dir * t_mid

    # 5. Convert to Grid Indices
    p_grid = p_world + vec3(0.5, 0.5, 0.5)
    ix = int(p_grid.x * grid_size)
    iy = int(p_grid.y * grid_size)
    iz = int(p_grid.z * grid_size)

    # 6. Return Success and Indices
    return True, (ix, iy, iz), p_world
```

**Edge Cases and Pitfalls:**
- If `hoverInfo.hasHit` is false after the ray-box intersection, the function returns false, meaning no voxel interaction occurs even if the ray hits the bounding box [S2, unofficial].
- Precision issues may occur at the boundaries of the volume if the ray grazes the box; the midpoint strategy helps mitigate but does not eliminate all edge cases.
- Division by zero is avoided by checking for near-zero direction components before slab intersection.

## Subtopics

### [Voxel Raycasting for Block Interaction](3780-voxel-raycasting-for-block-interaction.md)
A brute-force raycasting algorithm used to determine block intersection for player interaction, such as breaking or placing blocks in a voxel world. It differs from its parent, [Ray-Box Intersection for Interaction](2638-ray-box-intersection-for-interaction.md), which uses analytical intersection tests, by using discrete sampling; and from its child, [Voxel Block Interaction (Raycast Hit Detection)](3912-voxel-block-interaction-raycast-hit-detection.md), which performs a localized search around a known hit location rather than stepping from the origin.

## Operating characteristics

## Background

## Sources
[S1] tzhazuma/Fluid_simulation, https://github.com/tzhazuma/Fluid_simulation, fetched 2026-10-26 (unofficial)
[S2] GitHub - saurlax/model_voxelizer: A 3D model voxelization tool that ..., https://github.com/saurlax/model_voxelizer, fetched 2026-10-30 (unofficial)
