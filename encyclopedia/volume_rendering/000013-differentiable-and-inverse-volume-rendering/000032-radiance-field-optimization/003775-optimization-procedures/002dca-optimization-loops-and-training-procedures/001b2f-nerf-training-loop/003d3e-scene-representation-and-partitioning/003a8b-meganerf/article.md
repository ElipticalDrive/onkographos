# MegaNeRF

Also called: Altitude-constrained Raycasting, Ellipsoid Partitioning, Mega-NeRF, Mega-NeRF Foreground/Background Decomposition

## Overview
Mega-NeRF is a framework for training large-scale 3D scenes (spanning $10^5$ to $10^6$ m²) that supports interactive human-in-the-loop fly-throughs, addressing challenges of modeling thousands of images with varying lighting, large model capacities, and fast rendering [S1]. It differs from siblings like [BlockNeRF](8581-blocknerf.md) and [NeRFPlayer](9440-nerfplayer.md) by using spatial partitioning primarily to enable data parallelism for scalable training, rather than just speeding up inference of a pre-trained single network [S1]. It improves upon [NeRF++](https://arxiv.org/abs/2010.07492) by replacing the unit sphere foreground with an altitude-constrained ellipsoid and partitioning training pixels by geometric visibility [S1].

## How it works
**Inputs:** A set of $N$ training images with known camera poses (position $\mathbf{c}_i$, orientation, intrinsics) and, optionally, camera altitude measurements $h_i$.
**Outputs:** A set of $K$ independent NeRF submodules $\{ \mathcal{M}_k \}_{k=1}^K$, each associated with a spatial cell $C_k$.

**Data Structures:**
1.  **Spatial Partitioning:** The scene is decomposed into $K$ spatial cells $C_k$ via spatial Voronoi partitioning. The centroids of these cells are fixed at a common height $z_{centroid}$ to account for small altitude variations relative to latitude/longitude differences [S1].
2.  **Submodules:** Each cell $C_k$ has an associated MLP $\mathcal{M}_k$ that predicts density $\sigma$ and color $\mathbf{c}$ for points within that cell.
3.  **Pixel Partitioning:** A mapping $P: \text{Pixels} \rightarrow \{1, \dots, K\}$ assigns each training pixel $p$ from image $i$ to the set of cells $C_k$ whose volume intersects the ray cast from camera $i$ through $p$ [S1].

**Algorithm:**

1.  **Scene Decomposition:**
    Define $K$ spatial cells $C_k$ covering the scene volume. The foreground volume is modeled as an ellipsoid $\mathcal{E}$ that tightly encloses all camera poses and relevant foreground detail, replacing the unit sphere used in NeRF++ [S1]. The background volume is the complement of $\mathcal{E}$ [S1].

2.  **Geometric Clustering (Data Partitioning):**
    For each training image $i$ with pixel $p$:
    a. Cast a ray $\mathbf{r}_{i,p}$ from the camera center through pixel $p$.
    b. Identify the set of cells $\mathcal{K}_{i,p} = \{ k \mid \mathbf{r}_{i,p} \cap C_k \neq \emptyset \}$ [S1].
    c. Assign pixel $p$ to the training set of each submodule $\mathcal{M}_k$ for all $k \in \mathcal{K}_{i,p}$ [S1].
    *Note:* This reduces the size of each submodule's training set by approximately 10x compared to using all pixels [S1].

3.  **Foreground/Background Decomposition:**
    The scene is split into a foreground volume (inside $\mathcal{E}$) and a background volume (outside $\mathcal{E}$). Both are modeled by separate Mega-NeRFs [S1].
    *   **Foreground:** Uses the ellipsoid $\mathcal{E}$ for bounds.
    *   **Background:** Uses the same 4D outer volume parameterization and raycasting formulation as NeRF++ [S1].

4.  **Altitude-Constrained Raycasting (Inference):**
    During rendering, for a ray $\mathbf{r}$:
    a. Determine the entry and exit points of the ray within the scene bounds.
    b. Use camera altitude measurements to constrain sampling bounds. Terminate rays near ground level to avoid querying underground regions [S1].
    c. Sample points along the ray. For each sample point $\mathbf{x}$, identify the cell $C_k$ containing $\mathbf{x}$.
    d. Query the corresponding submodule $\mathcal{M}_k$ for $\sigma(\mathbf{x})$ and $\mathbf{c}(\mathbf{x})$ [S1].

5.  **Parallel Training:**
    Train each submodule $\mathcal{M}_k$ independently and in parallel using its partitioned pixel data [S1].

6.  **Interactive Rendering (Temporal Coherence):**
    To ensure near-interactive view generation, exploit temporal coherence by caching occupancy and color values from nearby previous views [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_mega_nerf(images, cells):
    """
    images: List of dicts with keys 'pose', 'intrinsics', 'pixels'
    cells: List of spatial cell definitions (e.g., Voronoi regions)
    """
    # Step 1: Partition pixels to cells
    pixel_to_cells = {}
    for img in images:
        for pixel in img.pixels:
            ray = cast_ray(img.pose, img.intrinsics, pixel)
            # Find cells intersected by the ray
            intersecting_cells = [c for c in cells if ray_intersects_cell(ray, c)]
            pixel_to_cells[pixel] = intersecting_cells

    # Step 2: Initialize submodules
    submodules = {c.id: init_mlp() for c in cells}
    
    # Step 3: Parallel Training
    # In practice, this is distributed across workers
    for cell in cells:
        # Get pixels relevant to this cell
        relevant_pixels = [p for p, cs in pixel_to_cells.items() if cell.id in cs]
        
        # Train submodule on its specific data
        train_module(submodules[cell.id], relevant_pixels)
        
    return submodules

def render_mega_nerf(camera_pose, submodules, cells, altitude_limit):
    """
    Render a single view using the trained submodules.
    """
    # Determine ray bounds using altitude constraint
    # Terminate rays near ground level
    ray_start, ray_end = compute_ray_bounds(camera_pose, altitude_limit)
    
    # Sample points along the ray
    samples = sample_points(ray_start, ray_end, num_samples=256)
    
    # Initialize accumulation variables
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_alpha = 0.0
    weights = []
    
    for i, point in enumerate(samples):
        # Find the cell containing this point
        cell = find_containing_cell(point, cells)
        if cell is None:
            continue
            
        # Query the specific submodule for this cell
        sigma, color = submodules[cell.id].forward(point)
        
        # Standard volume rendering accumulation
        # ... (standard NeRF alpha compositing) ...
        
    return accumulated_color
```

**Parameters:**
*   Number of cells $K$: Not given in the sources.
*   Ellipsoid parameters: Tightly encloses camera poses [S1].
*   Altitude limit: Determined by ground level measurements [S1].

**Edge Cases and Pitfalls:**
*   **Underground Sampling:** Without altitude constraints, rays may query underground regions, wasting computation [S1].
*   **Pre-training Infeasibility:** Unlike DeRF or KiloNeRF, Mega-NeRF cannot start from a pre-trained single network because training on all data at this scale is prohibitive [S1].

## Subtopics

### Ellipsoid Partitioning
Uses an ellipsoid instead of a unit sphere for the foreground volume to achieve tighter bounds on the region of interest [S1].

### Altitude-Constrained Raycasting
Uses camera altitude measurements to constrain ray sampling bounds, terminating rays near ground level to avoid querying underground regions [S1].

### Geometric Clustering
A method to partition training images (pixels) into different NeRF submodules based on visibility, enabling data parallelism [S1].

## Operating characteristics
*   **Time Complexity:** Training is accelerated by 3x compared to baseline approaches [S1]. Rendering achieves a 40x speedup over conventional NeRF rendering [S1][S3].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Improves PSNR by 12% compared to baseline approaches on existing datasets [S1]. Achieves 40x acceleration compared to traditional NeRF rendering while remaining within 0.8 dB in PSNR quality [S1][S3].
*   **Scaling:** Scales to scenes 1000x larger than prior work, from 150,000 to over 1,300,000 m² per scene [S1].
*   **Strengths:** Enables data parallelism for scalable training; supports interactive fly-throughs; handles large-scale urban environments.
*   **Limitations:** Requires geometric clustering preprocessing; relies on accurate camera altitude measurements for optimal performance.
*   **When to use:** Large-scale urban scenes requiring interactive fly-throughs where training a single NeRF is infeasible.
*   **When not to use:** Small-scale scenes where standard NeRF is sufficient; scenes without reliable altitude data.

## Background
Mega-NeRF was introduced in 2022 by [S1] to address the challenges of modeling large-scale 3D scenes from drone footage for search-and-rescue applications. It builds on [NeRF++](https://arxiv.org/abs/2010.07492) by improving the foreground/background decomposition and introduces geometric clustering for data parallelism [S1]. It differs from [BlockNeRF](8581-blocknerf.md) and [KiloNeRF](https://arxiv.org/abs/2106.00985) by using partitioning for training scalability rather than inference speedup of a pre-trained model [S1].

## Sources
[S1] Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly- Throughs, https://doi.org/10.1109/cvpr52688.2022.01258, fetched 2026-10-26
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S3] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
