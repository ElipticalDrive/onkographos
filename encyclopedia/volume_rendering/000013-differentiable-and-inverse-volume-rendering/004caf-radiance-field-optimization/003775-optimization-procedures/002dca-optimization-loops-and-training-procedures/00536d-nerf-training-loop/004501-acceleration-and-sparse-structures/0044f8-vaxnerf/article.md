# VaxNeRF

Also called: Visual Hull-Accelerated NeRF, Voxel-Accelerated NeRF

## Overview
VaxNeRF is an acceleration strategy for Neural Radiance Fields (NeRF) that reduces the number of neural network queries during training and inference by restricting ray sampling to the **visual hull** $V$ [S1]. The visual hull is defined as the maximum volume compatible with object silhouettes from a multi-view image collection [S1]. Unlike the original NeRF, which samples points over the entire scene bounding box using a two-stage coarse/fine sampling strategy, VaxNeRF uses a single MLP with uniform sampling restricted to voxels inside $V$ [S1]. This approach typically yields a 2-8x speedup in training time without degrading rendering quality [S1].

## How it works
VaxNeRF accelerates the NeRF optimization loop by culling ray samples that fall outside the object's silhouette volume. The method consists of a preprocessing step to construct the visual hull and a modified training loop that filters samples based on this hull.

### Inputs and Outputs
*   **Inputs:**
    *   A set of $N$ input views $\{I_i\}_{i=1}^N$, each with a known camera model $P_i$ and a coarse foreground-background segmentation mask $M_i$ [S1].
    *   Scene bounding box parameters (implicitly defined by the voxel grid resolution $D$ and extent) [S1].
*   **Outputs:**
    *   A voxel grid representation of the visual hull $V$ [S1].
    *   A trained NeRF model (single MLP) capable of rendering novel views [S1].

### Data Structures
**Visual Hull Grid:**
*   A 3D voxel grid of size $D \times D \times D$ [S1].
*   Each voxel stores an integer count indicating occupancy [S1].
*   Default resolution: $D = 400$ [S1].
*   Memory footprint: $\sim 70$ MB for $D=400$ [S1].

**Auxiliary Grids (for construction):**
*   Two grids, $G_1$ and $G_2$, both of size $D \times D \times D$, initialized to 0 [S1].
    *   $G_1$: Counts intersections from foreground silhouettes [S1].
    *   $G_2$: Counts intersections from the full camera frustum [S1].

### Parameters
*   **Grid Resolution ($D$):** Side length of the voxel grid. Default: 400 [S1].
*   **Samples per Ray ($N$):** Number of uniformly distributed points sampled per ray. Suggested values: 600 or 800 [S1].
*   **Batch Size:** Number of rays processed per training step. Default: 1024 [S1].
*   **Dilation:** A moderate dilation is applied to the foreground masks $M_i$ or the resulting voxel grid to prevent object boundary leakage [S1].

### Visual Hull Construction
The visual hull $V$ is constructed by back-projecting foreground silhouettes into 3D space [S1]. The algorithm uses two auxiliary grids to ensure the hull is contained within the camera frustums of all views.

1.  Initialize $G_1$ and $G_2$ as $D \times D \times D$ grids of zeros [S1].
2.  For each input view $I_i$ with camera model $P_i$ and mask $M_i$:
    a.  **Foreground Projection:** Cast rays from all foreground pixels in $M_i$ using $P_i$. For each ray, discretize it into $D$ points within the depth limits. Increment $G_1$ for every voxel containing at least one of these points [S1].
    b.  **Frustum Projection:** Cast rays from all pixels in $I_i$ (background and foreground) using $P_i$. For each ray, discretize it into $D$ points. Increment $G_2$ for every voxel containing at least one of these points [S1].
3.  The visual hull $V$ consists of all voxel positions $(x,y,z)$ where $G_1[x,y,z] \ge G_2[x,y,z]$ AND $G_1[x,y,z] > 0$ [S1].

*Note: The discretization of a ray into $D$ points is used to determine voxel intersection [S1].*

### Camera Setup and Ray Generation
To sample rays, the method defines external and internal camera parameters [S1].

**External Parameters ($M_{pose}$):**
Defined by spherical coordinates: azimuth $\theta$, elevation $\phi$, and radius $r$ [S1].
$$
M_{pose} = R_\theta R_\phi T_r
$$
Where:
*   $R_\theta$ is rotation around the y-axis by $\theta$:
    $$
    R_\theta = \begin{pmatrix} \cos \theta & 0 & -\sin \theta & 0 \\ 0 & 1 & 0 & 0 \\ \sin \theta & 0 & \cos \theta & 0 \\ 0 & 0 & 0 & 1 \end{pmatrix}
    $$
*   $R_\phi$ is rotation around the x-axis by $\phi$:
    $$
    R_\phi = \begin{pmatrix} 1 & 0 & 0 & 0 \\ 0 & \cos \phi & \sin \phi & 0 \\ 0 & -\sin \phi & \cos \phi & 0 \\ 0 & 0 & 0 & 1 \end{pmatrix}
    $$
*   $T_r$ is translation by radius $r$:
    $$
    T_r = \begin{pmatrix} 1 & 0 & 0 & 0 \\ 0 & 1 & 0 & 0 \\ 0 & 0 & 1 & r \\ 0 & 0 & 0 & 1 \end{pmatrix}
    $$
The ray origin $\mathbf{o}$ is the first 3 values of the last column of $M_{pose}$ [S1].

**Internal Parameters ($K$):**
$$
K = \begin{pmatrix} f & 0 & w/2 \\ 0 & f & h/2 \\ 0 & 0 & 1 \end{pmatrix}
$$
Where $f$ is focal length, and $w, h$ are image width and height [S1].

**Ray Direction:**
1.  Compute direction in camera frame $\mathbf{d}_{cam}$ for pixel $(x,y)$:
    $$
    \mathbf{d}_{cam} = \left( \frac{x - w/2}{f}, \frac{y - h/2}{f}, -1 \right)^\top
    $$
2.  Transform to world frame:
    $$
    \mathbf{d} = M_{pose}^{3\times3} \mathbf{d}_{cam}
    $$
    Where $M_{pose}^{3\times3}$ is the upper-left $3\times3$ submatrix of $M_{pose}$ [S1].

### Training Loop
VaxNeRF replaces the original NeRF's two-stage sampling with a single-stage uniform sampling restricted to the visual hull [S1].

1.  For each ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$:
2.  Sample $N$ points $\{x_i\}_{i=1}^N$ uniformly along the ray [S1].
3.  **Culling:** Discard any point $x_i$ that does not lie inside the visual hull $V$ (i.e., if the voxel containing $x_i$ is not part of $V$) [S1].
4.  Compute the rendered color $\mathbf{c}(\mathbf{r})$ using only the remaining points:
    $$
    \mathbf{c}(\mathbf{r}) = \sum_{i : x_i \in V} T_i \alpha_i \mathbf{c}_i
    $$
    Where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$, $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$, and $\mathbf{c}_i$ is the color predicted by the MLP at $x_i$ [S1].
5.  Compute loss against ground truth pixel color and backpropagate through the single MLP [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_visual_hull(views, D=400):
    """
    Constructs the visual hull V from multi-view masks.
    views: List of dicts with keys 'mask' (binary image), 'K' (3x3), 'M_pose' (4x4).
    D: Grid resolution.
    Returns: 3D numpy array V of shape (D, D, D) with boolean values.
    """
    G1 = np.zeros((D, D, D), dtype=np.int32)
    G2 = np.zeros((D, D, D), dtype=np.int32)
    
    # Define scene bounds for discretization. 
    # Assuming the voxel grid covers the bounding box of the scene.
    # The exact bounds are not specified, but rays are discretized into D points.
    # We assume a standard range [-1, 1] or similar normalized space mapped to D.
    # For implementation, one must map ray t-values to voxel indices.
    
    for view in views:
        mask = view['mask']
        K = view['K']
        M_pose = view['M_pose']
        
        # Get foreground pixel coordinates
        fg_pixels = np.argwhere(mask > 0)
        
        # Process Foreground Rays (Update G1)
        for y, x in fg_pixels:
            ray_cam = get_ray_direction_cam(x, y, K)
            ray_world = transform_ray_to_world(ray_cam, M_pose)
            # Discretize ray into D points and mark voxels in G1
            mark_voxels(ray_world, D, G1)
            
        # Process All Rays (Update G2)
        # Iterate over all pixels in the image
        h, w = mask.shape
        for y in range(h):
            for x in range(w):
                ray_cam = get_ray_direction_cam(x, y, K)
                ray_world = transform_ray_to_world(ray_cam, M_pose)
                # Discretize ray into D points and mark voxels in G2
                mark_voxels(ray_world, D, G2)
                
    # Visual Hull V: G1 >= G2 AND G1 > 0
    V = (G1 >= G2) & (G1 > 0)
    return V

def get_ray_direction_cam(x, y, K):
    """
    Computes ray direction in camera coordinates.
    x, y: Pixel coordinates.
    K: 3x3 intrinsic matrix.
    Returns: 3D vector.
    """
    f = K[0, 0]
    w = K.shape[1] * 2 # Approximation based on K definition, or pass w/h explicitly
    h = K.shape[0] * 2
    # K definition: [[f, 0, w/2], [0, f, h/2], [0, 0, 1]]
    # Principal point is (w/2, h/2)
    px = K[0, 2]
    py = K[1, 2]
    
    dx = (x - px) / f
    dy = (y - py) / f
    return np.array([dx, dy, -1.0])

def transform_ray_to_world(ray_cam, M_pose):
    """
    Transforms ray direction and origin to world coordinates.
    """
    # Origin is last column of M_pose (first 3 elements)
    origin = M_pose[:3, 3]
    
    # Direction is M_pose[:3, :3] @ ray_cam
    rot = M_pose[:3, :3]
    direction = rot @ ray_cam
    
    return {'origin': origin, 'direction': direction}

def mark_voxels(ray, D, grid):
    """
    Discretizes a ray into D points and increments grid voxels.
    """
    origin = ray['origin']
    direction = ray['direction']
    direction = direction / np.linalg.norm(direction)
    
    # Define depth limits. Not explicitly given, but typically [-1, 1] or scene bounds.
    # Assuming normalized bounding box [-1, 1] for simplicity in this generic impl.
    t_min = -1.0
    t_max = 1.0
    
    # Sample D points
    t_values = np.linspace(t_min, t_max, D)
    
    for t in t_values:
        point = origin + t * direction
        # Map point to voxel index
        # Assuming grid covers [-1, 1] in each dimension
        # voxel_idx = (point + 1) * (D / 2)
        # This mapping depends on the specific coordinate system of the grid.
        # The source says "discretized into D points per ray within the depth limits".
        # We assume a standard mapping.
        idx = point_to_voxel_index(point, D)
        if is_valid_index(idx, D):
            grid[idx] += 1

def point_to_voxel_index(point, D):
    # Placeholder for specific coordinate-to-voxel mapping.
    # Standard: index = floor((point - min_bound) / voxel_size)
    # Assuming bounds [-1, 1]
    voxel_size = 2.0 / D
    idx = ((point + 1.0) / voxel_size).astype(int)
    return tuple(idx)

def is_valid_index(idx, D):
    return all(0 <= i < D for i in idx)

def train_vaxnerf(V, views, N=600, batch_size=1024):
    """
    Training loop for VaxNeRF.
    """
    model = NeRF_MLP() # Single MLP
    optimizer = Adam(model.parameters())
    
    for step in range(num_steps):
        # Sample batch of rays
        rays = sample_rays(views, batch_size)
        
        targets = get_ground_truth_colors(rays)
        
        # Forward pass
        colors = []
        for ray in rays:
            # Sample N points uniformly
            t_values = np.linspace(ray.t_min, ray.t_max, N)
            points = ray.origin + t_values[:, None] * ray.direction
            
            # Cull points outside Visual Hull
            valid_indices = []
            for i, point in enumerate(points):
                if is_inside_visual_hull(point, V):
                    valid_indices.append(i)
            
            if not valid_indices:
                # If no points inside, handle gracefully (e.g., background color)
                colors.append(background_color)
                continue
                
            valid_points = points[valid_indices]
            
            # Query MLP
            # NeRF MLP takes (position, direction) -> (sigma, color)
            # This part assumes standard NeRF query logic
            sigmas, rgbs = query_nerf(model, valid_points, ray.direction)
            
            # Volume Rendering
            # Compute weights for valid points
            # T_i = exp(-sum(sigma_j * delta_j))
            # alpha_i = 1 - exp(-sigma_i * delta_i)
            # color = sum(T_i * alpha_i * c_i)
            
            rendered_color = volume_render(valid_points, ray, sigmas, rgbs)
            colors.append(rendered_color)
            
        loss = compute_loss(colors, targets)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

def is_inside_visual_hull(point, V):
    idx = point_to_voxel_index(point, V.shape[0])
    if is_valid_index(idx, V.shape[0]):
        return V[idx]
    return False
```

## Operating characteristics
*   **Speed:** VaxNeRF trains 2-8x faster than the original NeRF [S1]. The speedup factor depends on the visual hull size relative to the total scene volume; larger objects (e.g., palace, ship) yield factors of 4 or less, while finer objects (e.g., steamtrain) yield factors up to 8 [S1].
*   **Memory:** The voxel grid for $D=400$ requires $\sim 70$ MB, compared to $\sim 7$ MB for the NeRF MLP weights [S1].
*   **Quality:** No performance degradation in rendering quality (PSNR) compared to original NeRF [S1]. In some cases (e.g., steamtrain), slight PSNR improvements were observed, though convergence issues were noted in specific long-duration tests [S1].
*   **Construction Time:** Visual hull construction takes $\le 10$ seconds for typical input collections [S1].
*   **Strengths:** Simple implementation requiring minimal code changes to existing NeRF pipelines; does not require fine segmentation masks; single MLP simplifies the architecture [S1].
*   **Limitations:** Requires coarse foreground-background segmentation masks for all input views; memory usage increases with grid resolution; less effective for scenes where the object occupies a large portion of the bounding box (reduced culling benefit) [S1].

## Background
VaxNeRF was proposed by Kondo et al. in 2021 [S1]. It builds on the concept of the visual hull, a classic technique in multi-view 3D reconstruction [S1]. The method was evaluated on the Synthetic-NeRF and Synthetic-NSVF datasets [S1]. An open-source JAX implementation is available [S1]. The method is cited in the context of accelerating NeRF optimization by reducing the search space [S1].

## Sources
[S1] Accelerating NeRF with the Visual Hull, https://doi.org/10.5201/ipol.2024.553, fetched 2026-10-26
