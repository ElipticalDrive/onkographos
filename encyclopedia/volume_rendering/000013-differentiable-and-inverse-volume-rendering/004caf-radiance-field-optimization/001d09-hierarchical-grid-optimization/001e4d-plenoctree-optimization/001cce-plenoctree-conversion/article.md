# PlenOctree Conversion

Also called: NeRF-SH to PlenOctree conversion, PlenOctree baking

## Overview
PlenOctree Conversion is a pipeline that transforms a trained NeRF-SH model into a sparse octree representation (PlenOctree) to enable real-time rendering [S1, unofficial]. It solves the performance bottleneck of conventional NeRFs, which require dense sampling and neural network inference for every ray step, by replacing online queries with direct lookups of density and spherical harmonic coefficients [S1, unofficial]. Unlike naive NeRF rendering, which cannot cache color values due to their dependency on viewing direction, PlenOctrees store these dependencies explicitly, allowing for acceleration without sacrificing free-viewpoint quality [S1, unofficial].

## How it works
**Inputs:** A trained NeRF-SH model (a neural network that outputs density $\sigma$ and spherical harmonic coefficients for RGB radiance directly, eliminating the need for a view-direction input to the network) [S1, unofficial].
**Outputs:** A sparse octree data structure where each leaf node stores a density value $\sigma$ and a set of spherical harmonic (SH) coefficients representing view-dependent color [S1, unofficial].

**Data Structures:**
The PlenOctree is an octree structure.
*   **Internal Nodes:** Standard octree pointers to children.
*   **Leaf Nodes:**
    *   `density`: Float value $\sigma$ (volume density).
    *   `sh_coeffs`: Array of spherical harmonic coefficients for RGB channels. The number of coefficients depends on the SH degree used during NeRF-SH training (typically degree 3 or 4 for real-time applications, though exact degree is not specified in the primary source [S1, unofficial]).

**Steps:**
1.  **Evaluation:** Evaluate the NeRF-SH model to obtain $\sigma$ values on a uniformly spaced 3D grid. The grid is automatically scaled to tightly fit the scene content [S1, unofficial].
    *   *Implementation Note:* The grid resolution is determined by pre-evaluating $\sigma$ on a larger grid and finding the bounding box of all points with $\sigma \ge \tau_a$ (where $\tau_a$ is a small alpha threshold) [S1, unofficial].
2.  **Filtering:** Render alpha maps for all training views using the voxel grid derived from the evaluation step. Track the maximum ray weight $1 - \exp(-\sigma_i \delta_i)$ at each voxel, where $\delta_i$ is the segment length within the voxel [S1, unofficial].
    *   Eliminate voxels whose maximum ray weights are lower than a threshold $\tau_w$ [S1, unofficial]. This filtering by maximum ray weight eliminates non-visible voxels, which is superior to naively thresholding by $\sigma$ at each point [S1, unofficial].
3.  **Octree Construction:** Construct the octree containing the remaining voxels [S1, unofficial].
4.  **Sampling (Baking):** For each remaining voxel (leaf candidate), sample $N=256$ random points within the voxel volume [S1, unofficial].
5.  **Coefficient Aggregation:** Compute the mean of the density and SH coefficients obtained from the 256 sampled points. Set the associated leaf of the octree to these mean values to reduce aliasing [S1, unofficial].

**Parameters:**
*   $\tau_w$: Threshold for voxel weight (maximum ray weight) to determine which voxels are retained. Typical value not given in sources [S1, unofficial].
*   $N$: Number of random samples per voxel for anti-aliasing. Value: 256 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convert_nerf_sh_to_plenoctree(nerf_sh_model, training_views):
    """
    Converts a trained NeRF-SH model to a PlenOctree.
    
    Args:
        nerf_sh_model: A neural network that takes (x, y, z) and returns (sigma, sh_coeffs).
        training_views: List of camera poses and intrinsics for training images.
        
    Returns:
        plen_octree: A sparse octree structure with leaves containing density and SH coefficients.
    """
    # Step 1: Evaluation
    # Determine bounding box by finding points with sigma >= tau_a on a coarse grid
    # For implementation, assume a function get_scene_bounds() exists or is derived from data
    bounds = get_scene_bounds(nerf_sh_model, tau_a=1e-4) # [G] tau_a is a small alpha threshold
    
    # Define a uniform 3D grid covering the bounds
    # Resolution is not explicitly given, but must be fine enough to capture geometry
    grid_resolution = 256 # [G] Typical initial resolution; adjust based on scene size
    grid_points = generate_uniform_grid(bounds, grid_resolution)
    
    # Evaluate NeRF-SH on grid points
    grid_sigma = []
    for point in grid_points:
        sigma, _ = nerf_sh_model(point)
        grid_sigma.append(sigma)
        
    # Step 2: Filtering
    # Render alpha maps for all training views to compute max ray weight per voxel
    # Assume voxel_size is derived from grid_resolution and bounds
    voxel_size = (bounds.max - bounds.min) / grid_resolution
    
    # Initialize max weights for all voxels to 0
    max_ray_weights = np.zeros(grid_sigma.shape)
    
    for view in training_views:
        # Render ray through the voxel grid for this view
        rays = generate_rays(view, grid_resolution)
        for i, ray in enumerate(rays):
            # Intersect ray with voxel grid to get segments
            segments = intersect_ray_with_grid(ray, grid_points, voxel_size)
            
            # Compute ray weight for each segment
            # Weight = 1 - exp(-sigma * delta)
            # Track maximum weight encountered for each voxel index
            for seg in segments:
                voxel_idx = seg.voxel_index
                sigma_val = grid_sigma[voxel_idx]
                delta = seg.length
                weight = 1.0 - np.exp(-sigma_val * delta)
                if weight > max_ray_weights[voxel_idx]:
                    max_ray_weights[voxel_idx] = weight
                    
    # Threshold voxels
    tau_w = 0.01 # [G] Typical threshold for visibility; not given in sources
    active_voxel_indices = np.where(max_ray_weights > tau_w)[0]
    active_voxel_centers = grid_points[active_voxel_indices]
    
    # Step 3 & 4: Octree Construction and Sampling
    # Build octree from active voxel centers
    plen_octree = Octree()
    
    for idx in active_voxel_indices:
        voxel_center = grid_points[idx]
        voxel_bounds = get_voxel_bounds(voxel_center, voxel_size)
        
        # Sample 256 random points in the voxel
        num_samples = 256
        samples = np.random.uniform(voxel_bounds.min, voxel_bounds.max, size=(num_samples, 3))
        
        # Evaluate NeRF-SH at samples
        sample_sigmas = []
        sample_sh_coeffs = []
        for sample in samples:
            sigma, sh = nerf_sh_model(sample)
            sample_sigmas.append(sigma)
            sample_sh_coeffs.append(sh)
            
        # Average the values
        mean_sigma = np.mean(sample_sigmas)
        mean_sh_coeffs = np.mean(sample_sh_coeffs, axis=0)
        
        # Add leaf to octree
        plen_octree.insert_leaf(voxel_center, mean_sigma, mean_sh_coeffs)
        
    return plen_octree
```

## Subtopics

### Fourier PlenOctree
A method designed for human silhouette rendering using Shape-From-Silhouette, storing colors and densities on a coarse PlenOctree and using a Fourier Transform MLP [S3]. It constructs a coarse visual hull using sparse views predicted from a generalized NeRF and Shape-From-Silhouette, then densely samples colors and densities inside this hull to store on a coarse PlenOctree [S3]. After sampling dense views from the PlenOctree and applying transmissivity thresholding to eliminate most empty points, it generates new leaf densities and spherical harmonic (SH) color coefficients for remaining points and updates the PlenOctree [S3]. Finally, it uses a Fourier Transform MLP to extract Fourier coefficients of the density and SH color coefficients, which are fed into an inverse discrete Fourier transform to restore the SH coefficients and density [S3].

## Operating characteristics
*   **Time Complexity:** The full extraction process (evaluation, filtering, sampling) takes about 15 minutes for the Synthetic NeRF dataset [S1, unofficial].
*   **Memory Complexity:** Depends on the sparsity of the scene and the threshold $\tau_w$. The octree structure allows for significant memory savings compared to dense grids by only storing leaves for visible/occupied regions [S1, unofficial].
*   **Performance:** Enables real-time rendering of NeRFs by replacing neural network queries with direct lookups [S1, unofficial].
*   **Strengths:** Eliminates the need for network inference during rendering; handles view-dependent effects via SH coefficients; skips large empty voxels [S1, unofficial].
*   **Limitations:** Requires a pre-trained NeRF-SH model; conversion time is non-trivial (15 mins); quality depends on the SH degree and sampling density [S1, unofficial].
*   **When to use:** When real-time rendering of pre-trained NeRFs is required and the scene can be represented well by spherical harmonics.
*   **When not to use:** For scenes with very high-frequency view-dependent effects that require higher SH degrees (increasing storage) or for dynamic scenes where the NeRF needs to be updated frequently.

## Background
PlenOctree Conversion was proposed in the paper "PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs)" [S1, unofficial]. It builds on the NeRF-SH variant, which modifies the standard NeRF network to directly output SH coefficients, thereby eliminating the need for a view-direction input [S1, unofficial]. The method was designed to address the performance bottleneck of conventional NeRFs, which require dense sampling and neural network inference for every ray step [S1, unofficial].

## Sources
[S1] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
[S2] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
