# Mega-NeRF Spatial Partitioning

Also called: Cell-based NeRF, Spatial Data Parallelism

## Overview
Mega-NeRF Spatial Partitioning decomposes a large-scale scene into spatial cells, each managed by a separate neural network submodule, to enable parallel training and reduce memory consumption [S1]. Unlike monolithic NeRFs or other sparse methods that may use hash grids or octrees, this approach assigns distinct MLP weights to geometric regions, allowing the training dataset for each submodule to be pruned based on visibility, significantly accelerating convergence for virtual fly-throughs [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of calibrated camera poses (position and orientation), corresponding images, and a scene bounding volume.
*   **Output:** A collection of neural network submodules $\{f^n\}$ associated with centroids $\mathcal{N}$, and appearance embeddings $\{l^{(a)}\}$ for each image $a$.

**Data Structures:**
1.  **Centroids ($\mathcal{N}$):** A set of 3D points $n = (n_x, n_y, n_z)$ representing the centers of spatial cells.
2.  **Submodules ($f^n$):** For each centroid $n$, a Multi-Layer Perceptron (MLP) containing fully connected layers. The architecture mirrors the canonical NeRF:
    *   **Density Head:** Outputs scalar opacity $\sigma$.
    *   **Color Head:** Outputs RGB color $\mathbf{c}$.
    *   *Note:* The specific internal layer dimensions are not given in the sources, but they follow the standard NeRF topology described in the parent article [Parent, S1].
3.  **Appearance Embeddings ($l^{(a)}$):** A learnable vector associated with each input image $a$ to account for lighting variations [S1].
4.  **Trainset Partition ($\mathcal{T}_n$):** For each cell $n$, a subset of training pixels/rays assigned to that cell.

**Algorithm Steps:**

1.  **Centroid Initialization:**
    Tessellate the scene into a top-down 2D grid. Fix the height ($z$-coordinate) of all centroids to a single value, as altitude variance is small relative to latitude/longitude differences in the target scenes [S1].

2.  **Submodule Initialization:**
    Initialize a set of MLP weights $\{f^n\}$, one for each centroid $n \in \mathcal{N}$. Initialize appearance embeddings $\{l^{(a)}\}$ for each training image [S1].

3.  **Visibility Partitioning (Data Assignment):**
    For each training image $a$:
    a. Sample points along the camera rays corresponding to each pixel.
    b. Identify which spatial cells these rays intersect.
    c. Assign the pixel to the trainset $\mathcal{T}_n$ only for those intersecting cells $n$.
    d. Apply an overlap factor (typically 15%) between adjacent cells to minimize visual artifacts at boundaries [S1].

4.  **Spatial Data Pruning (Optional Post-Processing):**
    After a coarse training phase, refine the trainsets:
    a. Use the coarse geometry inferred by the NeRFs to identify occluders.
    b. Prune pixels/rays from cells if they are occluded by geometry inferred by other cells (e.g., if a wall is detected in cell $F$, prune rays from cells $A$ and $B$ that pass behind it) [S1].

5.  **Query and Rendering:**
    For a query point $\mathbf{x}$, viewing direction $\mathbf{d}$, and image $a$:
    a. Find the closest centroid:
    $$
    \mathbf{n} = \argmin_{n \in \mathcal{N}} \lVert \mathbf{n} - \mathbf{x} \rVert^2
    $$
    b. Query the corresponding submodule $f^\mathbf{n}$:
    $$
    \sigma = f^\mathbf{n}(\mathbf{x})
    $$
    $$
    \mathbf{c} = f^\mathbf{n}(\mathbf{x}, \mathbf{d}, l^{(a)})
    $$
    c. Render the pixel color using standard volumetric rendering integration along the ray [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_mega_nerf(scene_bounds, grid_resolution, num_images):
    """
    Initialize the spatial partitioning structure.
    """
    # Step 1: Centroid Initialization
    # Tessellate scene into a top-down 2D grid.
    # Fix height to a single value (e.g., ground level or average camera height).
    centroids = create_2d_grid(scene_bounds, grid_resolution)
    # centroids is an array of shape (N, 3)
    
    # Step 2: Submodule Initialization
    # Initialize MLP weights for each centroid.
    # Architecture: Density MLP + Color MLP (see Parent article for details).
    submodules = [init_neurf_mlp() for _ in range(len(centroids))]
    
    # Initialize appearance embeddings for each image.
    appearance_embeddings = [init_embedding() for _ in range(num_images)]
    
    return centroids, submodules, appearance_embeddings

def assign_pixels_to_cells(centroids, training_images, overlap_factor=0.15):
    """
    Step 3: Visibility Partitioning.
    Assign pixels to cells based on ray intersection.
    """
    cell_trainsets = {i: [] for i in range(len(centroids))}
    
    for img_idx, image in enumerate(training_images):
        rays = generate_rays(image.camera_pose, image.resolution)
        
        for pixel_idx, ray in enumerate(rays):
            # Sample points along the ray to determine intersection
            # In practice, this checks if the ray passes through the bounding volume of a cell
            intersecting_cells = find_intersecting_cells(ray, centroids, overlap_factor)
            
            for cell_idx in intersecting_cells:
                # Add (pixel_idx, image_idx) to the cell's trainset
                cell_trainsets[cell_idx].append((pixel_idx, img_idx))
                
    return cell_trainsets

def prune_trainsets(centroids, submodules, cell_trainsets, num_iterations=100):
    """
    Step 4: Spatial Data Pruning.
    Prune occluded rays after coarse training.
    """
    # Perform coarse training to get initial geometry
    train_coarse(centroids, submodules, cell_trainsets, num_iterations)
    
    pruned_trainsets = {}
    
    for cell_idx, trainset in cell_trainsets.items():
        pruned_set = []
        for pixel_idx, img_idx in trainset:
            ray = get_ray_from_image(training_images[img_idx], pixel_idx)
            
            # Check if ray is occluded by geometry from OTHER cells
            # This requires querying the density of other submodules along the ray
            is_occluded = False
            for other_cell_idx, other_submodule in enumerate(submodules):
                if other_cell_idx == cell_idx:
                    continue
                
                # Query density from the other cell's model
                # If density is high along the ray segment belonging to this cell,
                # it implies an occluder from the other cell's region.
                # Note: The exact implementation of "intervening occluder" check
                # is described as "details in Sec. A of the supplement" [S1].
                # [G] Placeholder for specific occlusion check logic.
                if check_occlusion(ray, other_submodule, centroids[other_cell_idx]):
                    is_occluded = True
                    break
            
            if not is_occluded:
                pruned_set.append((pixel_idx, img_idx))
        
        pruned_trainsets[cell_idx] = pruned_set
        
    return pruned_trainsets

def query_mega_nerf(centroids, submodules, appearance_embeddings, x, d, img_idx):
    """
    Step 5: Query and Rendering.
    """
    # Find closest centroid
    distances = np.linalg.norm(centroids - x, axis=1)
    closest_idx = np.argmin(distances)
    n = centroids[closest_idx]
    submodule = submodules[closest_idx]
    l_a = appearance_embeddings[img_idx]
    
    # Query submodule
    sigma = submodule.density_network(x)
    c = submodule.color_network(x, d, l_a)
    
    return sigma, c
```

**Parameters:**
*   **Overlap Factor:** 15% (used to minimize visual artifacts near cell boundaries) [S1].
*   **Centroid Height:** Fixed to a single value for all centroids [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** Without the overlap factor, discontinuities may appear at cell boundaries.
*   **Occlusion Logic:** The pruning step relies on a coarse understanding of geometry. Incorrect pruning can remove valid data, leading to holes or artifacts. The sources note that this is an "initial exploration" and details are in the supplement [S1].
*   **Altitude Variance:** The method assumes small altitude variance relative to horizontal differences. Scenes with significant vertical variation (e.g., multi-story buildings with distinct floors) may require adjustments to the centroid height strategy [S1].

## Subtopics

### Foreground/Background Decomposition
Mega-NeRF can subdivide the scene into a foreground volume (enclosing all camera poses) and a background volume, modeled with separate Mega-NeRFs, similar to NeRF++ [S1].

### Centroid Selection Methods
While the 2D grid is the primary method, k-means clustering and uncertainty-based partitioning were explored but found less practical [S1].

## Operating characteristics
*   **Training Speed:** Visibility partitioning reduces the submodule trainset size by 10x compared to the initial aggregate trainset [S1]. Additional spatial data pruning reduces it by a further factor of 2x [S1].
*   **Memory:** By decomposing the scene into cells with separate MLPs, memory usage is distributed. Each submodule only needs to store weights for its local region, and the trainset for each is smaller [S1].
*   **Scalability:** The method is designed for large-scale scenes (e.g., city blocks) where a single NeRF would be too slow or memory-intensive [S1].
*   **Strengths:** Parallelizable training, reduced data per submodule, handles large scenes efficiently.
*   **Limitations:** Requires careful handling of cell boundaries; assumes relatively flat terrain for the fixed-height centroid strategy; occlusion pruning adds complexity.

## Background
Mega-NeRF was introduced in 2022 by [S1] to address the scalability limitations of NeRF for large-scale virtual fly-throughs. It builds on the principles of differentiable volume rendering and positional encoding [Parent, S1]. The method draws inspiration from NeRF in the Wild for appearance embeddings [S1] and NeRF++ for foreground/background decomposition [S1].

## Sources
[S1] Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly-Throughs, https://doi.org/10.1109/cvpr52688.2022.01258, fetched 2026-10-26
