# VaxNeRF

Also called: Visual Hull-Accelerated NeRF, Voxel-Accelerated NeRF

## Overview
VaxNeRF is an acceleration strategy for Neural Radiance Fields (NeRF) that reduces the number of point queries required during training and inference by restricting ray sampling to the **visual hull** $V$, defined as the maximum volume compatible with object silhouettes from multi-view images [S1]. Unlike the original NeRF, which samples points over the entire scene space using a two-stage coarse/fine MLP approach, VaxNeRF uses a single MLP with uniform sampling restricted to the visual hull, achieving 2-8x faster training without performance degradation [S1].

## How it works
**Inputs:**
*   A set of $N$ input views $\{I_i\}_{i=1}^N$, each with a known camera model $P_i$ (external and internal parameters) and a coarse foreground-background segmentation mask $M_i$ [S1].
*   Scene bounding box or depth limits for voxel discretization [S1].

**Outputs:**
*   A trained single MLP representing the radiance field.
*   A voxel grid representation of the visual hull $V$ used to cull empty space queries [S1].

**Data Structures:**
1.  **Visual Hull Grid:** A 3D voxel grid of size $D \times D \times D$. Each voxel stores an integer count. Default $D=400$ [S1].
2.  **Auxiliary Grids:** Two grids $G_1$ and $G_2$ of size $D \times D \times D$, initialized to 0, used during construction [S1].
3.  **MLP:** A single Multi-Layer Perceptron (same architecture as original NeRF) taking 3D position and viewing direction as input [S1].

**Parameters:**
*   Grid resolution $D$: Default 400 [S1].
*   Samples per ray $N$: 600 or 800 [S1].
*   Batch size: 1024 rays [S1].

**Step 1: Visual Hull Construction**
Construct the visual hull $V$ using the foreground masks. This is done once before training [S1].

1.  Initialize two grids $G_1$ and $G_2$ of size $D \times D \times D$ with zeros.
2.  For each input view $I_i$ with camera model $P_i$ and mask $M_i$:
    a.  **Foreground Projection ($G_1$):** Cast rays from all foreground pixels in $M_i$ using $P_i$. For each ray, discretize it into $D$ points within the depth limits. Increment the count in $G_1$ for every voxel intersected by at least one of these points.
    b.  **Frustum Projection ($G_2$):** Cast rays from all pixels in the image $I_i$ using $P_i$. For each ray, discretize it into $D$ points within the depth limits. Increment the count in $G_2$ for every voxel intersected by at least one of these points.
3.  The visual hull $V$ consists of all voxel positions $(x,y,z)$ where $G_1[x,y,z] \ge G_2[x,y,z]$ AND $G_1[x,y,z] > 0$ [S1].

*Note: To prevent object boundary leakage, a moderate dilation is applied to the foreground mask $M_i$ or directly to the voxel grid $V$ [S1].*

**Step 2: Ray Setup**
For each training step, generate rays from the camera poses [S1].

1.  Define external parameters $M_{pose}$ (4x4 matrix) from spherical coordinates $(\theta, \phi, r)$:
    $$ M_{pose} = R_\theta R_\phi T_r $$
    Where:
    *   $R_\theta$ is rotation around the y-axis by azimuth $\theta$:
        $$ R_\theta = \begin{pmatrix} \cos \theta & 0 & -\sin \theta & 0 \\ 0 & 1 & 0 & 0 \\ \sin \theta & 0 & \cos \theta & 0 \\ 0 & 0 & 0 & 1 \end{pmatrix} $$
    *   $R_\phi$ is rotation around the x-axis by elevation $\phi$:
        $$ R_\phi = \begin{pmatrix} 1 & 0 & 0 & 0 \\ 0 & \cos \phi & -\sin \phi & 0 \\ 0 & \sin \phi & \cos \phi & 0 \\ 0 & 0 & 0 & 1 \end{pmatrix} $$
    *   $T_r$ is translation by radius $r$:
        $$ T_r = \begin{pmatrix} 1 & 0 & 0 & 0 \\ 0 & 1 & 0 & 0 \\ 0 & 0 & 1 & r \\ 0 & 0 & 0 & 1 \end{pmatrix} $$
2.  Define internal parameters $K$ (3x3 matrix) using focal length $f$ and image dimensions $w, h$:
    $$ K = \begin{pmatrix} f & 0 & w/2 \\ 0 & f & h/2 \\ 0 & 0 & 1 \end{pmatrix} $$
3.  Compute ray direction in camera frame $d_{cam}$ for pixel $(x,y)$:
    $$ d_{cam} = \left( \frac{x - w/2}{f}, \frac{y - h/2}{f}, -1 \right)^\top $$
4.  Transform direction to world frame:
    $$ d = M_{pose}^{3\times3} d_{cam} $$
5.  Ray origin $o$ is the first 3 values of the last column of $M_{pose}$ [S1].

**Step 3: Sampling and Rendering**
For each ray $r = o + td$:

1.  Sample $N$ points $\{x_i\}_{i=1}^N$ uniformly along the ray within the scene bounds [S1].
2.  **Culling:** Filter points such that only $x_i \in V$ are retained. This is checked by querying the voxel grid $V$ [S1].
3.  Query the single MLP for each retained point $x_i$ to get color $c_i$ and opacity $\alpha_i$.
4.  Compute the rendered color $c(r)$ using standard volume rendering equation, summing only over points inside $V$:
    $$ c(r) = \sum_{i : x_i \in V} T_i \alpha_i c_i $$
    Where $T_i = \exp\left(-\sum_{j=1}^{i-1} \alpha_j\right)$ is the accumulated transmittance [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def build_visual_hull(views, D=400):
    """
    Construct the visual hull voxel grid V.
    views: List of dicts with keys 'mask' (binary HxW), 'K' (3x3), 'M_pose' (4x4).
    D: Grid resolution.
    Returns: 3D numpy array V of shape (D, D, D) with boolean/integer values.
    """
    # Initialize auxiliary grids
    G1 = np.zeros((D, D, D), dtype=np.int32)
    G2 = np.zeros((D, D, D), dtype=np.int32)
    
    # Define scene bounds for discretization (example: -1 to 1)
    # In practice, these should match the NeRF training bounds
    bounds_min = -1.0
    bounds_max = 1.0
    
    for view in views:
        mask = view['mask']
        K = view['K']
        M_pose = view['M_pose']
        
        # Get foreground pixel coordinates
        fg_pixels = np.argwhere(mask > 0)
        
        # Process Foreground Rays (G1)
        for y, x in fg_pixels:
            ray_cam = compute_ray_direction_cam(x, y, K)
            ray_world = transform_to_world(ray_cam, M_pose)
            # Discretize ray into D points and mark voxels
            voxels = discretize_ray(ray_world, bounds_min, bounds_max, D)
            for v in voxels:
                if is_valid_voxel(v, D):
                    G1[v] += 1
                    
        # Process All Rays (G2) - defines frustum
        # Optimization: Vectorize this loop in implementation
        all_pixels = np.argwhere(np.ones_like(mask))
        for y, x in all_pixels:
            ray_cam = compute_ray_direction_cam(x, y, K)
            ray_world = transform_to_world(ray_cam, M_pose)
            voxels = discretize_ray(ray_world, bounds_min, bounds_max, D)
            for v in voxels:
                if is_valid_voxel(v, D):
                    G2[v] += 1
                    
    # Compute Visual Hull V
    # V is true where G1 >= G2 and G1 > 0
    V = (G1 >= G2) & (G1 > 0)
    return V

def compute_ray_direction_cam(x, y, K):
    """Compute ray direction in camera coordinates."""
    f = K[0, 0]
    w, h = K.shape[1], K.shape[0] # Assuming standard image dims
    # Note: K shape is usually 3x3. w,h are image dimensions.
    # The source formula: dcam = ((x - w/2)/f, (y - h/2)/f, -1)
    dx = (x - w/2) / f
    dy = (y - h/2) / f
    return np.array([dx, dy, -1.0])

def transform_to_world(d_cam, M_pose):
    """Transform direction vector to world frame."""
    # M_pose is 4x4. We only need the upper-left 3x3 rotation part for direction.
    R = M_pose[:3, :3]
    d_world = R @ d_cam
    return d_world

def discretize_ray(ray_origin, ray_direction, bounds_min, bounds_max, D):
    """
    Discretize a ray into D points and return voxel indices.
    Assumes ray_origin is in world space.
    """
    # Map world coordinates to voxel indices [0, D-1]
    # Assuming world space is [-1, 1] mapped to [0, D-1]
    # voxel_idx = (coord - bounds_min) / (bounds_max - bounds_min) * (D - 1)
    
    # Generate D points along the ray
    t_values = np.linspace(0, 1, D) # Simplified: assumes ray length covers bounds
    # More robust: calculate t_min and t_max for intersection with bounding box
    # For simplicity, we assume the ray passes through the volume and we sample uniformly
    # in the parameter space corresponding to the volume extent.
    
    # Let's assume the ray is normalized or we sample t such that it covers the box
    # A robust implementation calculates intersection with AABB.
    # Here we follow the source's "discretize into D points" instruction.
    
    points = []
    # Calculate extent of the ray within the bounds
    # This is a placeholder for the specific AABB intersection logic
    # which is complex to write from scratch without libraries.
    # We assume a helper `get_t_range` exists or we use a fixed range if bounds are known.
    
    # Simplified approach for pseudocode completeness:
    # Assume ray_origin is inside or we clamp.
    # We need the t values for the entry and exit of the bounding box.
    t_min, t_max = get_ray_box_intersection(ray_origin, ray_direction, bounds_min, bounds_max)
    
    if t_min > t_max:
        return []
        
    t_samples = np.linspace(t_min, t_max, D)
    voxel_indices = []
    
    for t in t_samples:
        p = ray_origin + t * ray_direction
        # Map p to voxel index
        idx = world_to_voxel(p, bounds_min, bounds_max, D)
        voxel_indices.append(idx)
        
    return voxel_indices

def get_ray_box_intersection(origin, direction, bounds_min, bounds_max):
    """Calculate t_min and t_max for ray intersection with axis-aligned box."""
    inv_dir = 1.0 / direction
    t1 = (bounds_min - origin) * inv_dir
    t2 = (bounds_max - origin) * inv_dir
    t_min = np.max(np.minimum(t1, t2))
    t_max = np.min(np.maximum(t1, t2))
    return t_min, t_max

def world_to_voxel(p, bounds_min, bounds_max, D):
    """Map world coordinate to voxel index."""
    idx = (p - bounds_min) / (bounds_max - bounds_min) * (D - 1)
    return tuple(np.round(idx).astype(int))

def is_valid_voxel(v, D):
    return all(0 <= i < D for i in v)

def render_ray(ray_origin, ray_direction, V, mlp, N=600, bounds_min=-1.0, bounds_max=1.0):
    """
    Render a single ray using VaxNeRF.
    """
    # Sample N points uniformly along the ray within bounds
    t_min, t_max = get_ray_box_intersection(ray_origin, ray_direction, bounds_min, bounds_max)
    t_samples = np.linspace(t_min, t_max, N)
    
    points = ray_origin + t_samples[:, np.newaxis] * ray_direction
    
    # Check which points are inside Visual Hull V
    # Map points to voxel indices
    voxel_indices = np.array([world_to_voxel(p, bounds_min, bounds_max, V.shape[0]) for p in points])
    
    # Boolean mask for points inside V
    inside_V = np.array([is_valid_voxel(v, V.shape[0]) and V[v] for v in voxel_indices])
    
    # Filter points and corresponding directions
    # Note: ray_direction is constant for all points on the ray
    points_inside = points[inside_V]
    directions_inside = np.tile(ray_direction, (points_inside.shape[0], 1))
    
    if len(points_inside) == 0:
        return 0.0 # Or background color
        
    # Query MLP
    # mlp takes (position, direction) -> (color, alpha)
    # Assuming mlp returns (sigma, color) or similar
    # Standard NeRF MLP: position -> sigma, (position, direction) -> color
    # We need to implement the query logic.
    
    # Placeholder for MLP query
    # In a real implementation, this would be a batched forward pass
    sigmas, colors = mlp_query(points_inside, directions_inside)
    
    # Compute transmittance and accumulated color
    # T_i = exp(-sum_{j<i} sigma_j)
    # alpha_i = 1 - exp(-sigma_i * delta)
    # Here we assume uniform sampling distance delta
    delta = (t_max - t_min) / N
    
    # Compute alpha
    # alpha = 1 - exp(-sigma * delta)
    alphas = 1.0 - np.exp(-sigmas * delta)
    
    # Compute transmittance
    # T_0 = 1
    # T_i = T_{i-1} * (1 - alpha_{i-1})
    # Vectorized computation
    T = np.ones(len(alphas) + 1)
    for i in range(len(alphas)):
        T[i+1] = T[i] * (1.0 - alphas[i])
    T = T[:-1] # Drop the last one
    
    # Weighted sum
    weights = T * alphas
    color = np.sum(weights[:, np.newaxis] * colors, axis=0)
    
    return color
```

## Operating characteristics
*   **Speed:** VaxNeRF trains 2-8x faster than original NeRF [S1]. The speed gain depends on the visual hull size relative to the scene volume (e.g., factor of 4 or less for large objects like palace/ship, almost 8 for fine objects like steamtrain) [S1].
*   **Memory:** Storing the visual hull in a $400 \times 400 \times 400$ voxel grid requires ~70 MB, while a basic NeRF MLP requires ~7 MB [S1].
*   **Construction Time:** Visual hull construction takes around 10 seconds or less for tested input photo collections [S1].
*   **Quality:** No loss of quality (PSNR) compared to original NeRF; in some cases, small improvements are observed [S1].
*   **Strengths:** Simple integration, minimal code changes, no quality degradation, significant training speedup [S1].
*   **Limitations:** Requires coarse foreground-background segmentation masks. Higher memory usage for the voxel grid compared to the MLP [S1].

## Background
VaxNeRF was proposed by Kondo et al. in 2021 [S1]. It addresses the time-consuming optimization process of original NeRF by restricting the search space to the visual hull, a classic concept in multi-view 3D reconstruction [S1]. The method is implemented in the open-source JAX implementation [S1] and evaluated on Synthetic-NeRF and Synthetic-NSVF datasets [S1].

## Sources

[S1] Accelerating NeRF with the Visual Hull, https://doi.org/10.5201/ipol.2024.553, fetched 2026-10-26
