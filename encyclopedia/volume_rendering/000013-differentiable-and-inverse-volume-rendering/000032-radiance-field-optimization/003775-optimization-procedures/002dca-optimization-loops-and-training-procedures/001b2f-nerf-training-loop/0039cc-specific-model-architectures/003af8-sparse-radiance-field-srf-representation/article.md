# Sparse Radiance Field (SRF) Representation


## Overview
A Sparse Radiance Field (SRF) is a voxel-based representation of a 3D scene or object that stores volume density and view-dependent radiance colors in a sparse grid structure [S1]. It serves as a discrete, efficient alternative to the continuous multi-layer perceptron (MLP) used in standard NeRF architectures, enabling fast optimization and direct mesh extraction via algorithms like Marching Cubes [S1]. Unlike dense voxel grids, SRFs utilize a Coordinate Format (COO) to store only non-empty voxels, significantly reducing memory usage while maintaining high-fidelity rendering capabilities [S1].

## How it works
**Inputs and Outputs:**
The SRF representation takes a 3D scene defined by posed images as input and outputs a sparse set of voxel features. The primary output is the SRF data structure itself, which can be rendered into 2D images using standard volume rendering equations or converted into 3D meshes [S1].

**Data Structures:**
The SRF is defined as a voxel grid of dimension $1+d$, where $d$ is the dimension of the radiance color features and 1 represents the volume density [S1].
*   **Grid Dimensions:** The grid has size $H$ in each of the three spatial dimensions ($X, Y, Z$). The total potential size is $H \times H \times H \times (1+d)$ [S1].
*   **Sparse Storage (COO):** Due to high sparsity, the SRF is stored in Coordinate Format (COO) as a set of $M$ tuples [S1]:
    $$ X_{\text{non-empty}} = \{(c_m, f_m)\}_{m=1}^M $$
    where:
    *   $c_m = (i, j, k)$ are positive integer coordinates ($i, j, k \in \mathbb{Z}^+$) identifying the voxel location [S1].
    *   $f_m \in \mathbb{R}^{d+1}$ is the feature vector for that voxel [S1].
    *   The feature vector $f_m$ consists of the density $\alpha_{i,j,k} \in \mathbb{R}$ and the radiance colors $\rho_{i,j,k} \in \mathbb{R}^d$ [S1].
    *   The ordering of the $M$ tuples is arbitrary [S1].
    *   The sparsity is defined as $1 - \frac{M}{H^3}$ [S1].

**Parameters:**
*   **Resolution ($H$):** The grid size in each dimension. Typical values used in the SPARF dataset are $H \in \{32, 128, 512\}$ [S1].
*   **Radiance Dimension ($d$):** The number of features per voxel for color.
    *   For view-dependent SRF using 4 spherical harmonics (SH) factors per RGB channel: $d = 3 \times 4 = 12$ [S1].
    *   For fixed RGB colors (no view dependence): $d = 3 \times 1 = 3$ [S1].
    *   Note: Using 4 SH components reduces optimization space by 40% and time by 10% compared to 9 SH components, while maintaining similar PSNR [S1]. Using raw RGB instead of SH reduces PSNR by ~1 dB but saves 80% space and 20% time [S1].

**Algorithm Steps:**
1.  **Initialization:** Define the grid resolution $H$ and radiance dimension $d$. Initialize an empty list for COO tuples.
2.  **Optimization:** Optimize the SRF parameters (densities and radiance colors) by minimizing the difference between rendered images and ground truth images. This is typically done using volume rendering [S1].
    *   *Note:* The specific optimization loop details are covered in the parent article [Specific Model Architectures](8585-specific-model-architectures.md).
3.  **Sparse Storage:** After optimization, iterate through the $H \times H \times H$ grid. For each voxel $(i, j, k)$:
    *   If the voxel is "non-empty" (i.e., has significant density or was updated during optimization), append the tuple $(c_m, f_m)$ to the COO list, where $c_m = (i, j, k)$ and $f_m = [\alpha_{i,j,k}, \rho_{i,j,k}]$.
    *   If the voxel is empty, skip it.
4.  **Rendering (Inference):** To render a view:
    *   Cast rays through the scene.
    *   For each ray, query the COO structure for voxels intersected by the ray.
    *   Apply the volume rendering equation:
        $$ C(r) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i, \quad T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right), \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i) $$
        where $\sigma_i$ is the density $\alpha_{i,j,k}$ and $\mathbf{c}_i$ is the radiance color $\rho_{i,j,k}$ [S1].
5.  **Mesh Extraction:** To extract a mesh, apply the Marching Cubes algorithm to the density field $\alpha_{i,j,k}$ of the SRF [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_srf_coo(grid_size_h, radiance_dim_d, voxels):
    """
    Converts a dense voxel grid to Sparse Radiance Field COO format.
    
    Args:
        grid_size_h (int): Resolution H of the grid.
        radiance_dim_d (int): Dimension d of radiance colors.
        voxels (dict): Dictionary mapping (i, j, k) tuples to feature arrays [density, radiance...].
                       Only non-empty voxels should be present in this dict.
                       
    Returns:
        list: List of tuples (coordinates, features) representing the SRF.
    """
    coo_list = []
    
    # Iterate over all non-empty voxels provided in the input dictionary
    for coords, features in voxels.items():
        # coords is expected to be (i, j, k)
        # features is expected to be an array of size d+1: [density, r0, ..., rd-1]
        
        # Validate coordinates are positive integers
        i, j, k = coords
        if not (isinstance(i, int) and isinstance(j, int) and isinstance(k, int)):
            raise ValueError("Coordinates must be integers")
            
        # Validate feature dimension
        if len(features) != radiance_dim_d + 1:
            raise ValueError(f"Feature dimension mismatch: expected {radiance_dim_d + 1}, got {len(features)}")
            
        # Append to COO list
        # The ordering is arbitrary as per source [S1]
        coo_list.append((coords, features))
        
    return coo_list

def render_srf_ray(coo_list, ray_origin, ray_direction, grid_size_h, radiance_dim_d):
    """
    Renders a single ray through a Sparse Radiance Field.
    
    Args:
        coo_list (list): The SRF COO representation.
        ray_origin (np.array): 3D origin of the ray.
        ray_direction (np.array): Normalized 3D direction of the ray.
        grid_size_h (int): Resolution H.
        radiance_dim_d (int): Dimension d.
        
    Returns:
        np.array: Rendered color (RGB) and accumulated alpha.
    """
    # 1. Determine voxel bounds for the ray
    # This requires mapping ray coordinates to voxel indices
    # For simplicity, we assume a helper function `get_voxels_intersected_by_ray`
    # which returns a list of (i, j, k) indices sorted by distance along the ray.
    intersected_voxel_indices = get_voxels_intersected_by_ray(
        ray_origin, ray_direction, grid_size_h
    )
    
    # 2. Create a lookup map for fast feature retrieval
    voxel_map = {coords: features for coords, features in coo_list}
    
    accumulated_transmittance = 1.0
    accumulated_color = np.zeros(3) # Assuming RGB output
    
    # 3. Iterate through intersected voxels
    for idx in intersected_voxel_indices:
        if idx in voxel_map:
            features = voxel_map[idx]
            density = features[0]
            radiance = features[1:] # Shape (d,)
            
            # Calculate step size delta (distance between voxel boundaries)
            # This depends on the specific ray segment within the voxel
            delta = get_voxel_step_size(ray_origin, ray_direction, idx, grid_size_h)
            
            # Calculate alpha (opacity) for this segment
            # alpha = 1 - exp(-sigma * delta)
            alpha = 1.0 - np.exp(-density * delta)
            
            # Weight the radiance by the probability of stopping here
            # weight = T * alpha
            weight = accumulated_transmittance * alpha
            
            # Accumulate color
            # Note: If d > 3, radiance might need to be converted to RGB
            # Assuming radiance is already RGB or d=3 for this example
            accumulated_color += weight * radiance[:3]
            
            # Update transmittance
            accumulated_transmittance *= (1.0 - alpha)
            
            # Early exit if fully opaque
            if accumulated_transmittance < 1e-5:
                break
                
    return accumulated_color, 1.0 - accumulated_transmittance
```

**Edge Cases and Pitfalls:**
*   **Low-Density Voxels:** Non-empty voxels may have low densities that do not significantly affect volume rendering but contain random color information. This can confuse the network during training [S1].
*   **Vanishing Gradients:** High sparsity (typically ~99%) can lead to vanishing gradients [S1].
*   **Coordinate Misalignment:** Misalignment between input SRF coordinates and output SRF coordinates, often caused by sparse convolutional strides and pruning, can further harm gradients and affect the learning process [S1].
*   **Small Errors:** Small errors in predicting densities or radiance colors can result in large distortions in the rendered images [S1].

## Operating characteristics
*   **Memory Complexity:** The memory usage is proportional to the number of non-empty voxels $M$, rather than the total grid size $H^3$. This makes it highly efficient for scenes with sparse geometry [S1].
*   **Time Complexity:** Rendering time depends on the number of intersected non-empty voxels along each ray. Optimization time is generally faster than dense MLP-based NeRFs due to the structured nature of the voxel grid and the ability to use sparse convolutions [S1].
*   **Performance:** The SPARF dataset demonstrates that SRFs can be optimized efficiently using large numbers of images (e.g., 400 views per shape) to reduce the number of iterations required while maintaining high PSNR [S1].
*   **Strengths:**
    *   Direct mesh extraction via Marching Cubes [S1].
    *   Efficient storage for sparse scenes [S1].
    *   Scalable to large datasets (e.g., 1 million SRFs) [S1].
*   **Limitations:**
    *   Sensitive to sparsity-induced gradient issues [S1].
    *   Requires careful handling of low-density voxels to avoid noise [S1].
    *   Resolution is discrete; increasing resolution requires re-optimization or multi-resolution strategies [S1].

## Background
The Sparse Radiance Field (SRF) representation was introduced in the context of the SPARF (Sparse Radiance Field) dataset and SuRFNet architecture [S1]. It builds upon the concept of Plenoxels, which uses spherical harmonics for view-dependent radiance in a voxel grid [S1]. The SPARF dataset, containing one million SRFs across multiple resolutions (32, 128, 512), was created to facilitate large-scale learning of 3D sparse radiance fields from few input images [S1]. The representation allows for the collection of a massive dataset of 3D shapes from 13 different classes, enabling robust novel view synthesis and 3D generation tasks [S1].

## Sources
[S1] SPARF: Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images, https://doi.org/10.1109/iccvw60793.2023.00315, fetched 2026-10-26
