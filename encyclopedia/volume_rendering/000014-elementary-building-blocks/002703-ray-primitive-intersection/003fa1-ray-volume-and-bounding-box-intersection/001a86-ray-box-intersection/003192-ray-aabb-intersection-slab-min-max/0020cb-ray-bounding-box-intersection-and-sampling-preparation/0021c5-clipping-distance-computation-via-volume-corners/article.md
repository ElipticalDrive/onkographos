# Clipping Distance Computation via Volume Corners

Also called: IJK to RAS Corner Projection, Volume Bounding Box Clipping

## Overview
Computes the near and far clipping distances for ray marching by projecting the eight corners of the volume bounding box onto the camera view direction, then taking the minimum and maximum of these projected distances [S1, unofficial]. This method ensures that all sample points generated along a ray fall within the volume's spatial extent, preventing out-of-bounds memory access and unnecessary computation outside the volume [S1, unofficial].

## How it works
**Inputs:**
- Camera matrix $M_{cam}$ (4x4) [S1, unofficial].
- Volume bounding box dimensions in IJK space: $I, J, K$ (number of voxels along each axis) [S1, unofficial].
- IJK-to-RAS transformation matrix $M_{ijk2ras}$ (4x4) [S1, unofficial].
- Margin parameter $\delta$ (default 30.0 mm) [S1, unofficial].

**Outputs:**
- Near clipping distance $d_{near}$ [S1, unofficial].
- Far clipping distance $d_{far}$ [S1, unofficial].

**Data Structures:**
- Camera matrix $M_{cam}$: 4x4 matrix. If the norm of the translation column (last column, first 3 rows) is less than 1.0, it is treated as a world-to-camera matrix and inverted to obtain the camera-to-world matrix $M_{world}$. Otherwise, it is used directly as $M_{world}$ [S1, unofficial].
- Volume corners in IJK space: 8 points $(i, j, k)$ where $i \in \{0, I-1\}$, $j \in \{0, J-1\}$, $k \in \{0, K-1\}$ [S1, unofficial].
- Volume corners in RAS space: 8 points $\mathbf{p}_{ras}^{(n)}$ for $n=0..7$ [S1, unofficial].

**Steps:**
1. **Extract Camera Pose:** Determine the camera position $\mathbf{c}$ and view direction $\mathbf{v}$ from $M_{cam}$ [S1, unofficial]. If $\|M_{cam}[0:3, 3]\| < 1.0$, invert $M_{cam}$ to get $M_{world}$ [S1, unofficial]. Extract $\mathbf{c}$ and $\mathbf{v}$ from $M_{world}$ and normalize $\mathbf{v}$ [S1, unofficial].
2. **Generate IJK Corners:** Create the 8 corners of the volume bounding box in IJK space using indices $0$ and $I-1$, $0$ and $J-1$, and $0$ and $K-1$ [S1, unofficial].
3. **Transform to RAS Space:** For each corner $\mathbf{p}_{ijk} = (i, j, k, 1)^T$, compute $\mathbf{p}_{ras} = M_{ijk2ras} \cdot \mathbf{p}_{ijk}$ and take the first three components to get the 3D position in RAS space [S1, unofficial].
4. **Project Corners:** For each RAS corner $\mathbf{p}_{ras}^{(n)}$, compute the vector $\mathbf{d}^{(n)} = \mathbf{p}_{ras}^{(n)} - \mathbf{c}$ and project this vector onto the normalized view direction $\mathbf{v}$ to get the scalar distance $t^{(n)} = \mathbf{d}^{(n)} \cdot \mathbf{v}$ [S1, unofficial].
5. **Compute Clipping Distances:** Set $t_{min} = \min_n(t^{(n)})$ and $t_{max} = \max_n(t^{(n)})$. Set $d_{near} = t_{min} - \delta$ and $d_{far} = t_{max} + \delta$. Clamp $d_{near}$ to a minimum of 0.1 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_clipping_distances(camera_matrix, ijk2ras, volume_dims, margin=30.0):
    """
    Compute near and far clipping distances for volume ray marching.
    Expects camera_matrix and ijk2ras as list of lists (4x4).
    volume_dims is a tuple (I, J, K).
    """
    I, J, K = volume_dims

    # Step 1: Extract Camera Pose
    # Get translation column (last column, first 3 rows)
    translation = [camera_matrix[0][3], camera_matrix[1][3], camera_matrix[2][3]]
    translation_norm = math.sqrt(sum(t * t for t in translation))

    # Invert if world-to-camera (norm < 1.0), else use directly
    # A full 4x4 inverse routine is required; here we assume a helper `invert_4x4` exists.
    if translation_norm < 1.0:
        camera_matrix_world = invert_4x4(camera_matrix)
    else:
        camera_matrix_world = camera_matrix

    # Extract camera position
    cam_pos = [camera_matrix_world[0][3], camera_matrix_world[1][3], camera_matrix_world[2][3]]

    # Extract view direction (typically -Z axis of the world matrix)
    view_dir = [-camera_matrix_world[0][2], -camera_matrix_world[1][2], -camera_matrix_world[2][2]]
    v_norm = math.sqrt(sum(v * v for v in view_dir))
    view_dir = [v / v_norm for v in view_dir]

    # Step 2: Generate IJK Corners
    corners_ijk = []
    for i in [0, I - 1]:
        for j in [0, J - 1]:
            for k in [0, K - 1]:
                corners_ijk.append([float(i), float(j), float(k)])

    # Step 3: Transform to RAS Space
    corners_ras = []
    for p in corners_ijk:
        x, y, z = p
        # Homogeneous multiplication: p_ras = M * [x, y, z, 1]^T
        wx = ijk2ras[0][0] * x + ijk2ras[0][1] * y + ijk2ras[0][2] * z + ijk2ras[0][3]
        wy = ijk2ras[1][0] * x + ijk2ras[1][1] * y + ijk2ras[1][2] * z + ijk2ras[1][3]
        wz = ijk2ras[2][0] * x + ijk2ras[2][1] * y + ijk2ras[2][2] * z + ijk2ras[2][3]
        corners_ras.append([wx, wy, wz])

    # Step 4: Project Corners
    distances = []
    for p in corners_ras:
        dx = p[0] - cam_pos[0]
        dy = p[1] - cam_pos[1]
        dz = p[2] - cam_pos[2]
        t = dx * view_dir[0] + dy * view_dir[1] + dz * view_dir[2]
        distances.append(t)

    # Step 5: Compute Clipping Distances
    t_min = min(distances)
    t_max = max(distances)

    d_near = t_min - margin
    d_far = t_max + margin
    d_near = max(d_near, 0.1)

    return d_near, d_far

def invert_4x4(m):
    """Placeholder for a standard 4x4 matrix inverse implementation."""
    # In practice, use a linear algebra library or a full cofactor expansion.
    # Returns the inverted matrix.
    return m
```

**Parameters:**
- `margin`: Safety margin added to near/far distances in mm. Default value is 30.0. Positive values expand the clipping range [S1, unofficial].

**Edge Cases and Pitfalls:**
- If the camera is inside the volume, $t_{min}$ may be negative. The margin and clamping handle this, but $d_{near}$ is clamped to 0.1 to avoid zero or negative distances which could cause division by zero or sampling errors [S1, unofficial].
- Precision issues may arise if the volume is very large or the camera is very far. Using double precision for intermediate calculations may be necessary [G].
- The assumption that the view direction is the negative Z-axis of the world matrix is standard but should be verified against the specific camera convention used [G].

## Subtopics

### Volume Bounding Box Projection for Ray Marching
A vertex shader implementation that projects the world-space axis-aligned bounding box (AABB) of a fire volume to enable fragment shader ray marching [S2, unofficial]. It transforms volume-space corners (unit cube $[0,1]^3$) to world space using a `volume_to_world` matrix, then to clip space using view and projection matrices [S2, unofficial]. It outputs the world position of the corners for use in the fragment shader [S2, unofficial]. This variant uses GPU-based rendering pipelines with push constants and shaders [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ with respect to volume resolution, as it processes a fixed 8 corners [G].
- **Memory Complexity:** $O(1)$, storing only 8 corners and a few scalar values [G].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of rays if applied per-ray, but since the volume bounds are global, it can be computed once per volume and reused for all rays [G].
- **Strengths:** Simple, robust, and efficient. Ensures all samples are within the volume bounds [G].
- **Limitations:** Assumes the volume is axis-aligned in IJK space and convex. Does not account for internal structures or non-convex shapes [G].
- **When to use:** When a quick and reliable method to determine ray clipping bounds is needed for volume rendering [G].
- **When not to use:** For non-convex volumes or when more precise ray-box intersection is required [G].

## Background
This method is used in differentiable volume raycasting for digitally reconstructed radiographs (DRRs) [S1, unofficial]. The approach is based on projecting the volume bounding box corners onto the camera view direction to determine the integration range for ray marching [S1, unofficial].

## Sources
[S1] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
[S2] migaelstrydom/VoxelPhase, https://github.com/migaelstrydom/VoxelPhase, fetched 2026-10-28 (unofficial)
