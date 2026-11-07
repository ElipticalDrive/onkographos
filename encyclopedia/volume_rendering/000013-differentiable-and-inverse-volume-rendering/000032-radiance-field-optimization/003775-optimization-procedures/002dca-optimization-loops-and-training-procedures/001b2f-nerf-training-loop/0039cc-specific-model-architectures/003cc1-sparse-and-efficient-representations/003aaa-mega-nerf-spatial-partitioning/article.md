# Mega-NeRF Spatial Partitioning

Also called: Cell-based NeRF, Spatial Data Parallelism

## Overview
Mega-NeRF Spatial Partitioning addresses the memory and training time bottlenecks of standard NeRF when applied to large-scale, outdoor scenes by decomposing the scene into spatial cells, each managed by a separate MLP submodule trained in parallel [S1]. Unlike monolithic architectures, it assigns query points to the nearest centroid and prunes irrelevant training data based on visibility, enabling scalable construction of virtual fly-throughs [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of calibrated training images with poses; a scene bounding volume.
*   **Outputs:** A set of MLP submodules $\{f^n\}$, one per spatial cell $n \in \mathcal{N}$, and their associated appearance embeddings.

**Data Structures:**
*   **Centroids ($\mathcal{N}$):** A set of 3D points $n = (n_x, n_y, n_z)$ representing cell centers. In the standard implementation, these are arranged in a top-down 2D grid with a fixed height $z$ for all centroids [S1].
*   **Submodules ($f^n$):** Each $f^n$ is a Multi-Layer Perceptron (MLP) with fully connected layers, structurally similar to the original NeRF network [S1].
*   **Appearance Embeddings ($l^{(a)}$):** A vector associated with each input image $a$, used to account for lighting variations [S1].
*   **Trainsets:** Each submodule $f^n$ maintains a local training set of pixels/rays.

**Algorithm Steps:**

1.  **Centroid Initialization:**
    Tessellate the scene into a top-down 2D grid to determine centroid positions. Fix the $z$-coordinate of all centroids to a single value, justified by the small variance in altitude relative to latitude/longitude differences in the target scenes [S1].

2.  **Model Initialization:**
    Initialize a separate MLP submodule $f^n$ for each centroid $n \in \mathcal{N}$. Initialize appearance embeddings $l^{(a)}$ for each training image $a$ [S1].

3.  **Visibility Partitioning (Data Assignment):**
    For each training image $a$:
    a. Sample points along the camera rays corresponding to each pixel.
    b. Identify which spatial cells $n$ these rays intersect.
    c. Add the pixel to the local trainset of only those intersecting cells [S1].
    d. Apply an overlap factor (e.g., 15%) between adjacent cells to minimize boundary artifacts [S1].

4.  **Spatial Data Pruning (Optional Post-Processing):**
    After a coarse training phase, refine the trainsets:
    a. Use the coarse geometry inferred by the NeRFs to identify occluders.
    b. For each cell, prune pixels/rays that are occluded by geometry inferred by *other* cells (e.g., if cell $F$ infers a wall, prune rays from other cells that pass through that wall) [S1].

5.  **Query Evaluation:**
    Given a query point $\mathbf{x}$, direction $\mathbf{d}$, and appearance embedding $l^{(a)}$:
    a. Find the nearest centroid $n^*$:
    $$ n^* = \arg\min_{n \in \mathcal{N}} \| n - \mathbf{x} \|^2 $$
    b. Compute opacity $\sigma$ and color $\mathbf{c}$ using the submodule associated with $n^*$:
    $$ \sigma = f^{n^*}(\mathbf{x}) $$
    $$ \mathbf{c} = f^{n^*}(\mathbf{x}, \mathbf{d}, l^{(a)}) $$
    [S1]

6.  **Foreground/Background Decomposition:**
    Subdivide the scene into a foreground volume (enclosing all camera poses) and a background volume. Train separate Mega-NeRF instances for each, using the same spatial partitioning logic within each volume [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def mega_nerf_spatial_partitioning(training_images, scene_bounds, overlap_factor=0.15):
    """
    Initialize and configure Mega-NeRF spatial partitioning.
    
    Args:
        training_images: List of dicts with keys 'pose', 'image', 'id'.
        scene_bounds: Bounding box of the scene.
        overlap_factor: Fraction of cell width to overlap neighbors.
        
    Returns:
        centroids: Array of shape (N, 3) for cell centers.
        submodules: List of MLP models, one per centroid.
        appearance_embeddings: Dict mapping image_id to embedding vector.
        trainsets: Dict mapping centroid_index to list of (pixel_coords, image_id).
    """
    # Step 1: Centroid Initialization
    # Tessellate scene into top-down 2D grid.
    # Fix height z to a single value based on scene statistics.
    centroids = create_2d_grid_centroids(scene_bounds)
    N = len(centroids)
    
    # Step 2: Model Initialization
    submodules = [create_mlp_submodule() for _ in range(N)]
    appearance_embeddings = {img['id']: create_embedding_vector() for img in training_images}
    
    # Step 3: Visibility Partitioning
    trainsets = {i: [] for i in range(N)}
    
    for img in training_images:
        # Sample rays for each pixel in the image
        rays = sample_rays_from_image(img)
        
        for pixel_idx, ray in enumerate(rays):
            # Determine which cells the ray intersects
            intersecting_cells = find_intersecting_cells(ray, centroids, overlap_factor)
            
            # Assign pixel to trainsets of intersecting cells
            for cell_idx in intersecting_cells:
                trainsets[cell_idx].append({
                    'pixel': pixel_idx,
                    'image_id': img['id'],
                    'ray': ray
                })
                
    # Step 4: Spatial Data Pruning (Optional, after coarse training)
    # This step requires running a coarse optimization first.
    # trainsets = prune_occluded_rays(trainsets, submodules, centroids)
    
    return centroids, submodules, appearance_embeddings, trainsets

def query_mega_nerf(x, d, image_id, centroids, submodules, appearance_embeddings):
    """
    Query the radiance field at point x.
    """
    # Find nearest centroid
    distances = np.linalg.norm(centroids - x, axis=1)
    n_idx = np.argmin(distances)
    
    # Get submodule and appearance embedding
    module = submodules[n_idx]
    l_a = appearance_embeddings[image_id]
    
    # Compute opacity and color
    sigma = module.forward(x)
    c = module.forward(x, d, l_a)
    
    return sigma, c
```

**Parameters:**
*   **Overlap Factor:** 15% in experiments [S1].
*   **Centroid Height:** Fixed to a single value [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** Without overlap, visual discontinuities may appear at cell boundaries. The 15% overlap mitigates this [S1].
*   **Occlusion Handling:** Initial pixel assignment is based on camera positions, not geometry. Pruning is necessary to remove rays occluded by geometry inferred by other cells [S1].
*   **Altitude Variance:** The method assumes small altitude variance relative to horizontal scale. If this assumption fails, fixing centroid height may degrade performance [S1].

## Operating characteristics
*   **Memory:** Significantly reduced compared to monolithic NeRF because each submodule only stores weights for its local region and its local trainset [S1].
*   **Training Time:** Reduced due to parallel training of submodules and smaller trainsets per module [S1].
*   **Performance:** Visibility partitioning reduces submodule trainset size by 10x compared to the initial aggregate trainset [S1]. Additional visibility pruning reduces trainset sizes by a further factor of 2x [S1].
*   **Scalability:** Designed for large-scale scenes (e.g., North Pittsburgh) where standard NeRF would be infeasible [S1].
*   **Strengths:** Scalable construction, parallel training, handles lighting variations via appearance embeddings [S1].
*   **Limitations:** Requires careful handling of cell boundaries; assumes relatively flat terrain for fixed-height centroids [S1].

## Background
Mega-NeRF was introduced in 2022 by [S1] to enable scalable construction of large-scale NeRFs for virtual fly-throughs. It builds on the NeRF architecture [24] and incorporates ideas from NeRF in the Wild [21] (appearance embeddings) and NeRF++ [48] (foreground/background decomposition). The method was developed to address the limitations of standard NeRF in large outdoor environments.

## Sources
[S1] Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly-Throughs, https://doi.org/10.1109/cvpr52688.2022.01258, fetched 2026-10-26
