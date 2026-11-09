# VaxNeRF

Also called: Visual Hull-Accelerated NeRF, Voxel-Accelerated NeRF

## Overview
VaxNeRF is an acceleration strategy for Neural Radiance Fields (NeRF) that reduces the number of point queries required during training and inference by restricting ray sampling to the **visual hull** $V$, defined as the maximum volume compatible with object silhouettes from multi-view images [S1]. Unlike the original NeRF, which samples points over the entire scene bounding box using a coarse/fine two-stage sampling strategy, VaxNeRF uses a single MLP with uniform sampling restricted to voxels inside $V$ [S1]. This approach eliminates the need for adaptive sampling and reduces the search space, resulting in training speeds 2–8 times faster than original NeRF without performance degradation [S1].

## How it works
### Inputs and Outputs
*   **Inputs:**
    *   A set of $N$ input views $\{I_i\}_{i=1}^N$, each with a known camera model $P_i$ and a coarse foreground-background segmentation mask $M_i$ [S1].
    *   Scene bounding box parameters (implicitly defined by the voxel grid resolution $D$) [S1].
*   **Outputs:**
    *   A trained NeRF model (single MLP) capable of rendering novel views [S1].
    *   A voxel grid representation of the visual hull $V$ [S1].

### Data Structures
*   **Visual Hull Grid:** A 3D voxel grid of size $D \times D \times D$. Each voxel stores an integer count indicating occupancy. The default resolution is $D = 400$ [S1].
*   **Auxiliary Grids:** Two grids, $G_1$ and $G_2$, of size $D \times D \times D$, initialized to 0, used during the construction of $V$ [S1].
*   **Camera Parameters:**
    *   External parameter matrix $M_{pose}$ (4x4) [S1].
    *   Internal parameter matrix $K$ (3x3) [S1].

### Steps

#### 1. Visual Hull Construction
The visual hull $V$ is constructed by back-projecting foreground silhouettes from all input views into a 3D voxel grid [S1].

1.  Initialize two grids $G_1$ and $G_2$ of size $D \times D \times D$ with zeros [S1].
2.  For each input view $I_i$ with camera model $P_i$ and mask $M_i$:
    a.  **Foreground Projection ($G_1$):** Cast rays from all foreground pixels in $M_i$ using $P_i$. Increment the count in $G_1$ for all voxels intersected by these rays [S1].
    b.  **Frustum Projection ($G_2$):** Cast rays from all pixels in the image $I_i$ using $P_i$. Increment the count in $G_2$ for all voxels intersected by these rays [S1].
3.  The visual hull $V$ consists of all voxel positions where $G_1 \ge G_2$ AND $G_1 > 0$ [S1].

*Note: To prevent object boundary leakage, a moderate dilation is applied to the foreground masks $M_i$ or directly to the resulting voxel grid [S1]. Ray-voxel intersection is determined by discretizing each ray into $D$ points within depth limits and marking the voxel containing each point [S1].*

#### 2. Ray Generation
For novel view synthesis or training, rays are generated based on camera parameters [S1].

1.  **External Parameters:** Compute $M_{pose}$ from spherical coordinates (azimuth $\theta$, elevation $\phi$, radius $r$):
    $$
    M_{pose} = R_\theta R_\phi T_r
    $$
    Where:
    $$
    R_\theta = \begin{pmatrix} \cos \theta & 0 & -\sin \theta & 0 \\ 0 & 1 & 0 & 0 \\ \sin \theta & 0 & \cos \theta & 0 \\ 0 & 0 & 0 & 1 \end{pmatrix}, \quad
    R_\phi = \begin{pmatrix} 1 & 0 & 0 & 0 \\ 0 & \cos \phi & \sin \phi & 0 \\ 0 & -\sin \phi & \cos \phi & 0 \\ 0 & 0 & 0 & 1 \end{pmatrix}, \quad
    T_r = \begin{pmatrix} 1 & 0 & 0 & 0 \\ 0 & 1 & 0 & 0 \\ 0 & 0 & 1 & r \\ 0 & 0 & 0 & 1 \end{pmatrix}
    $$
    [S1]

2.  **Internal Parameters:** Define $K$ using focal length $f$, width $w$, and height $h$:
    $$
    K = \begin{pmatrix} f & 0 & w/2 \\ 0 & f & h/2 \\ 0 & 0 & 1 \end{pmatrix}
    $$
    [S1]

3.  **Ray Direction (Camera Frame):** For a pixel at coordinates $(x, y)$, compute the direction vector in the camera frame:
    $$
    d_{cam} = \left( \frac{x - w/2}{f}, \frac{y - h/2}{f}, -1 \right)^\top
    $$
    [S1]

4.  **Ray Direction (World Frame):** Transform $d_{cam}$ to the world frame using the upper-left 3x3 subset of $M_{pose}$:
    $$
    d = M_{pose}^{3\times3} d_{cam}
    $$
    [S1]

5.  **Ray Origin:** The origin $o$ is the first 3 values of the last column of $M_{pose}$ [S1].

#### 3. Training and Rendering
VaxNeRF uses a single MLP with uniform sampling, replacing the coarse/fine two-stage sampling of original NeRF [S1].

1.  For each ray $r$, sample $N$ points $\{x_i\}_{i=1}^N$ uniformly along the ray [S1].
2.  Filter points: Only include points $x_i$ that lie inside the visual hull $V$ (i.e., $x_i \in V$) [S1].
3.  Compute the rendered color $c(r)$ using the standard volume rendering equation, summing only over points inside $V$:
    $$
    c(r) = \sum_{i : x_i \in V} T_i \alpha_i c_i
    $$
    Where $T_i$ is the accumulated transmittance, $\alpha_i$ is the opacity, and $c_i$ is the color predicted by the MLP at $x_i$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_visual_hull(input_views, D=400):
    """
    Constructs the visual hull V from multi-view silhouettes.
    input_views: List of tuples (I_i, P_i, M_i)
    D: Grid resolution
    Returns: 3D numpy array V of shape (D, D, D)
    """
    G1 = np.zeros((D, D, D), dtype=np.int32)
    G2 = np.zeros((D, D, D), dtype=np.int32)
    
    for I_i, P_i, M_i in input_views:
        # Get foreground pixels from mask M_i
        foreground_pixels = get_foreground_pixels(M_i)
        
        # Update G1: Back-project foreground silhouettes
        for pixel in foreground_pixels:
            ray = cast_ray_from_pixel(pixel, P_i)
            voxels_intersected = discretize_ray_into_voxels(ray, D)
            for v in voxels_intersected:
                G1[v] += 1
                
        # Update G2: Back-project entire image frustum
        all_pixels = get_all_pixels(I_i.shape)
        for pixel in all_pixels:
            ray = cast_ray_from_pixel(pixel, P_i)
            voxels_intersected = discretize_ray_into_voxels(ray, D)
            for v in voxels_intersected:
                G2[v] += 1
                
    # Visual Hull V: Voxels where foreground count >= frustum count AND foreground count > 0
    V = (G1 >= G2) & (G1 > 0)
    return V

def generate_ray(x, y, w, h, f, M_pose):
    """
    Generates a ray origin and direction in world coordinates.
    """
    # Internal parameters
    K = np.array([[f, 0, w/2],
                  [0, f, h/2],
                  [0, 0, 1]])
                  
    # Ray direction in camera frame
    d_cam = np.array([(x - w/2)/f, (y - h/2)/f, -1.0])
    
    # Ray direction in world frame
    M_pose_3x3 = M_pose[:3, :3]
    d = M_pose_3x3 @ d_cam
    
    # Ray origin
    o = M_pose[:3, 3]
    
    return o, d

def render_vaxnerf(ray_origin, ray_direction, visual_hull, mlp, N=600, D=400):
    """
    Renders a single ray using VaxNeRF sampling.
    """
    # Uniform sampling along the ray
    # Assuming ray is clipped to scene bounds [t_min, t_max]
    t_values = np.linspace(t_min, t_max, N)
    points = ray_origin + t_values[:, None] * ray_direction
    
    # Filter points inside visual hull
    # Note: In practice, voxel indices are computed from points
    voxel_indices = points_to_voxel_indices(points, D)
    inside_hull = visual_hull[voxel_indices]
    
    # Select points inside hull
    valid_points = points[inside_hull]
    
    # Query MLP for density and color
    # sigma, color = mlp(valid_points)
    
    # Volume rendering integration
    # c(r) = sum(T_i * alpha_i * c_i) for i where x_i in V
    # ... implementation of standard NeRF integration ...
    pass
```

### Parameters
*   **Grid Resolution ($D$):** Default $D = 400$ [S1].
*   **Samples per Ray ($N$):** Suggested values $N = 600$ or $N = 800$ [S1].
*   **Batch Size:** Default 1024 rays per training step [S1].
*   **Training Steps:** Evaluated up to 512,000 steps [S1].

### Edge Cases and Pitfalls
*   **Object Boundary Leakage:** If the segmentation mask is too tight, parts of the object may be excluded from the visual hull, causing artifacts. A moderate dilation should be applied to the foreground masks $M_i$ or the voxel grid $V$ [S1].
*   **Ray Discretization:** The accuracy of voxel intersection depends on discretizing rays into $D$ points. Insufficient discretization may lead to incorrect hull construction [S1].
*   **Memory Usage:** The $400^3$ voxel grid requires approximately 70 MB of memory, which is significantly higher than the ~7 MB required for the NeRF MLP itself [S1].

## Operating characteristics
*   **Speed:** VaxNeRF trains 2–8 times faster than original NeRF [S1]. The speed gain depends on the ratio of the visual hull volume to the total scene volume (e.g., factor of 4 for large objects like palace/ship, almost 8 for fine objects like steamtrain) [S1].
*   **Quality:** No performance degradation in terms of PSNR compared to original NeRF; in some cases, small improvements are observed [S1].
*   **Memory:** Requires ~70 MB for the visual hull grid vs ~7 MB for the NeRF MLP [S1].
*   **Construction Time:** Visual hull construction takes around 10 seconds or less for typical input collections [S1].
*   **Strengths:** Simplicity of implementation (minimal code changes to NeRF), no need for coarse/fine MLPs, fast training [S1].
*   **Limitations:** Requires foreground-background segmentation masks for all input views [S1].

## Background
VaxNeRF was proposed by Kondo et al. in 2021 [S1]. It addresses the time-consuming optimization process of original NeRF by leveraging the classic multi-view 3D reconstruction concept of the visual hull [S1]. The method was evaluated on Synthetic-NeRF and Synthetic-NSVF datasets, demonstrating superior training speed and competitive PSNR compared to original NeRF, NSVF, and PlenOctrees [S1]. An open-source JAX implementation is available [S1].

## Sources
[S1] Accelerating NeRF with the Visual Hull, https://doi.org/10.5201/ipol.2024.553, fetched 2026-10-26
