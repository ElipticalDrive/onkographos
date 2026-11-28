# Binary Occupancy Grid Pruning

Also called: Grid-based Pruning, History Pruning

## Overview
Binary Occupancy Grid Pruning accelerates volume rendering by explicitly skipping free space, thereby minimizing the number of sample points for which hash grid features need to be queried and MLPs evaluated [S1]. It sets itself apart from its siblings by combining a coarse binary occupancy grid with two specific pruning strategies—Cylinder Pruning and Joint History- and Grid-based Pruning—to achieve high rendering speeds without model baking [S1]. Unlike [Distance Map Acceleration](972-distance-map-acceleration.md) which uses signed distances, or [MERF Real-Time Rendering](10175-merf-real-time-rendering.md) which uses different acceleration structures, this method relies on a binary mask derived from training history and dense sampling estimates [S1].

## How it works
The method builds on the shared principles of [Occupancy Grid and Map-Based Empty Space Skipping](3852-occupancy-grid-and-map-based-empty-space-skipping.md). It constructs a secondary binary occupancy grid to represent the volume, defines active regions based on density thresholds, and skips free space during ray traversal [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Capture rig geometry (approximated as cylinders) [S1].
    *   Training history (maximum density observed per voxel) [S1].
    *   Scene geometry (estimated via dense sampling) [S1].
    *   Pruning parameters: initial resolution $R_{init}=128^3$, start iteration $I_{start}=100,000$, frequency $F_{prune}=1,000$, max threshold $\alpha=0.2$ [S1].
*   **Outputs:**
    *   A binary occupancy grid where voxels are marked as `occupied` or `free` [S1].
    *   During rendering, a list of active intervals along each ray [G].

### Data Structures
*   **Binary Occupancy Grid:** A 3D array of boolean values.
    *   Layout: $G \in \{0, 1\}^{R \times R \times R}$, where $R$ is the current grid resolution [S1].
    *   Value `1` indicates the voxel is occupied (may contain geometry) [G].
    *   Value `0` indicates the voxel is free (can be skipped) [G].
*   **History Buffer:** A 3D array storing the maximum density observed for each voxel during training.
    *   Layout: $H \in \mathbb{R}^{R \times R \times R}$ [S1].
    *   Initialized to $0.0$ [G].
*   **Grid-Based Density Buffer:** A 3D array storing the maximum density estimated via dense sampling inside each voxel.
    *   Layout: $D \in \mathbb{R}^{R \times R \times R}$ [S1].
    *   Initialized to $0.0$ [G].

### Steps

1.  **Initialization:**
    *   Initialize the binary occupancy grid $G$ with resolution $R_{init} = 128^3$ [S1].
    *   Initialize $H$ and $D$ to zeros [G].
    *   Apply **Cylinder Pruning** (see Subtopics) to mark voxels inside the capture rig cylinders as `free` ($0$) [S1].

2.  **Training Phase (History Tracking):**
    *   During training, for each ray sample falling within a voxel $v$:
        *   Update the history buffer: $H[v] \leftarrow \max(H[v], \rho)$, where $\rho$ is the density contribution of the sample [S1].

3.  **Pruning Phase (Joint History- and Grid-based Pruning):**
    *   This phase begins after iteration $I_{start} = 100,000$ [S1].
    *   Every $F_{prune} = 1,000$ iterations:
        *   **Grid-based Estimation:** For each voxel $v$ in $G$:
            *   Evaluate a dense cubic grid inside voxel $v$ to estimate the maximum density [S1].
            *   Use the minimum step size possible inside the voxel based on the ray from the closest camera as a worst-case estimate for density calculation [S1].
            *   Update $D[v]$ with this estimated maximum density [G].
        *   **Threshold Calculation:**
            *   Calculate the current pruning threshold $\tau$ [S1].
            *   $\tau$ is annealed linearly from $0$ to $\alpha = 0.2$ over the course of the pruning phase [S1].
            *   Let $I_{current}$ be the current iteration. Let $I_{end}$ be the final iteration (or a predefined milestone).
            *   $\tau = \alpha \times \frac{I_{current} - I_{start}}{I_{end} - I_{start}}$ (clamped to $[0, \alpha]$) [G].
        *   **Voxel Pruning:**
            *   For each voxel $v$:
                *   If $H[v] < \tau$ AND $D[v] < \tau$:
                    *   Mark $G[v] = 0$ (free) [S1].
                *   Else:
                    *   Mark $G[v] = 1$ (occupied) [S1].

4.  **Upsampling:**
    *   At predefined iteration milestones, upsample the occupancy grid $G$ to a higher resolution to prune scenes more accurately [S1].
    *   The history and density buffers are also upsampled accordingly [G].

5.  **Rendering (Ray Traversal):**
    *   For each ray, query the binary occupancy grid $G$ to find the entry and exit points of active regions [G].
    *   Skip intervals where $G$ is `0` [S1].
    *   Evaluate the MLP and hash grid features only for samples within `1` voxels [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def ray_grid_traversal(ray_origin, ray_direction, grid, voxel_size, max_t):
    """
    Traverses a uniform binary occupancy grid along a ray using 3D DDA.
    Returns a list of (t_start, t_end) intervals for occupied voxels.
    """
    ox, oy, oz = ray_origin
    dx, dy, dz = ray_direction
    
    # Guard against zero direction components
    if dx == 0: dx = 1e-9
    if dy == 0: dy = 1e-9
    if dz == 0: dz = 1e-9

    # Initial grid cell indices
    cx = int(math.floor(ox / voxel_size))
    cy = int(math.floor(oy / voxel_size))
    cz = int(math.floor(oz / voxel_size))

    # Step direction per axis
    sx = 1 if dx > 0 else -1
    sy = 1 if dy > 0 else -1
    sz = 1 if dz > 0 else -1

    # Distance to next grid boundary along each axis
    t_max_x = ((cx + (1 if sx > 0 else 0)) * voxel_size - ox) / dx
    t_max_y = ((cy + (1 if sy > 0 else 0)) * voxel_size - oy) / dy
    t_max_z = ((cz + (1 if sz > 0 else 0)) * voxel_size - oz) / dz

    # Distance between consecutive boundaries
    t_delta_x = voxel_size / abs(dx)
    t_delta_y = voxel_size / abs(dy)
    t_delta_z = voxel_size / abs(dz)

    intervals = []
    t = 0.0

    while t < max_t:
        # Boundary check
        if not (0 <= cx < grid.shape[0] and 0 <= cy < grid.shape[1] and 0 <= cz < grid.shape[2]):
            break

        # Determine which axis boundary is reached next
        if t_max_x < t_max_y:
            if t_max_x < t_max_z:
                t_next = t_max_x
                cx += sx
            else:
                t_next = t_max_z
                cz += sz
        else:
            if t_max_y < t_max_z:
                t_next = t_max_y
                cy += sy
            else:
                t_next = t_max_z
                cz += sz

        # Record interval if the current cell is occupied
        if grid[cx, cy, cz]:
            intervals.append((t, t_next))

        t = t_next
        # Advance the t_max for the axis that was crossed
        if t_next == t_max_x: t_max_x += t_delta_x
        elif t_next == t_max_y: t_max_y += t_delta_y
        else: t_max_z += t_delta_z

    return intervals

def render_ray(ray_origin, ray_direction, grid, voxel_size, max_t, evaluate_density_color):
    """
    Renders a single ray using the occupancy grid for empty-space skipping.
    evaluate_density_color(pos) returns (density, color_rgb) at pos.
    """
    intervals = ray_grid_traversal(ray_origin, ray_direction, grid, voxel_size, max_t)
    color = [0.0, 0.0, 0.0]
    transmittance = 1.0

    for t_start, t_end in intervals:
        num_samples = max(1, int(math.ceil((t_end - t_start) / voxel_size)))
        step = (t_end - t_start) / num_samples
        
        for i in range(num_samples):
            t = t_start + i * step
            pos = (ray_origin[0] + t * ray_direction[0],
                   ray_origin[1] + t * ray_direction[1],
                   ray_origin[2] + t * ray_direction[2])
            
            density, color_contrib = evaluate_density_color(pos)
            alpha = 1.0 - math.exp(-density * step)
            
            color[0] += alpha * transmittance * color_contrib[0]
            color[1] += alpha * transmittance * color_contrib[1]
            color[2] += alpha * transmittance * color_contrib[2]
            transmittance *= (1.0 - alpha)
            
            if transmittance < 1e-4:
                break
                
    return color
```

### Parameters
*   $\alpha$: Maximum pruning threshold value. Typical value: $0.2$ [S1].
*   Initial pruning start iteration: $100,000$ [S1].
*   Pruning frequency: Every $1,000$ iterations [S1].
*   Initial grid resolution: $128^3$ [S1].

### Edge Cases and Pitfalls
*   **Precision:** The density estimation in grid-based pruning depends on the step size. Using a step size that is too large may underestimate density, leading to over-pruning [S1].
*   **Boundaries:** Cylinder pruning must ensure that voxels are *completely* inside the cylinder to be marked as free, to avoid cutting into valid geometry [S1].
*   **Degenerate Input:** If the scene has no geometry (e.g., pure black background), all voxels may be pruned, leading to empty renders. The threshold annealing helps mitigate this by being conservative early on [S1].

## Subtopics

### Joint History- and Grid-based Pruning
A conservative pruning strategy combining history-based pruning (tracking max density per voxel during training) with grid-based pruning (evaluating a dense cubic grid inside each voxel to estimate max density) to prune occupancy grid voxels [S1]. It is chosen when higher accuracy in pruning is needed compared to using only one method. It prunes grid voxels where both maximum densities (history and grid-based) fall below the current pruning threshold [S1]. The pruning threshold is annealed linearly from $0$ to $0.2$ [S1].

### Cylinder Pruning
A pruning strategy that approximates the capture rig geometry as a cylinder and marks all occupancy grid voxels completely inside any such cylinder as free space [S1]. It is used to initialize the occupancy grid based on known rig capture positions [S1]. It prevents the model from cheating using floaters in front of cameras, leading to more view-consistent models, and speeds up early stages of training [S1].

## Operating characteristics
*   **Time Complexity:** Pruning is performed periodically (every 1,000 iterations) and involves dense sampling inside each voxel, which is expensive. However, it is done offline or during training, not at render time. Rendering time is reduced by skipping free space [S1].
*   **Memory Complexity:** Requires storing the binary occupancy grid, history buffer, and density buffer. For a $128^3$ grid, this is $128^3 \times 3$ voxels. Upsampling increases memory usage [S1].
*   **Measured Performance:** The method enables native rendering on a Meta Quest Pro VR headset at dual 2K×2K eyebuffers at 72 FPS [S1].
*   **Strengths:** Maximizes rendering speed without model baking; prevents cheating with floaters; improves convergence speed in early training [S1].
*   **Limitations:** Requires training history; dense sampling for grid-based pruning is computationally expensive; initial coarse resolution may lead to inaccurate pruning [S1].
*   **When to use:** When high-performance rendering is required, especially on mobile/VR hardware, and training time is available for pruning [S1].
*   **When not to use:** When real-time adaptation to dynamic scenes is required, as pruning is based on static training history [S1].

## Background
Binary Occupancy Grid Pruning was introduced by Müller et al. in 2022 in the context of VR-NeRF [S1]. It was developed to address the high run-time cost of NeRF rendering in virtual reality, specifically for native rendering on headsets like the Meta Quest Pro [S1]. The method extends widely-adopted occupancy grid acceleration structures by adding novel pruning strategies [S1].

## Sources
[S1] VR-NeRF: High-Fidelity Virtualized Walkable Spaces, https://doi.org/10.1145/3610548.3618139, fetched 2026-10-28
