# Valley Cell-Based Clustering Transfer Function Design

Also called: Automated TF Design with Valley Cell-Based Clustering, VCC-TF

## Overview
Valley Cell-Based Clustering (VCC-TF) is an automatic method for generating 2D opacity transfer functions by decomposing feature density plots into topological "valley cells" and mapping them to Gaussian functions [S1]. It solves the problem of manually defining transfer functions for complex volumetric data by automatically identifying distinct features based on their separability in the feature space (e.g., value vs. gradient magnitude) [S1]. Unlike Binning-based Transfer Functions (BTF) which require user-specified bin counts and lack simplification guidelines, VCC-TF automatically determines the number of features and uses cell persistence to hierarchically simplify the classification, removing noise while preserving significant structures [S1].

## How it works
**Inputs:**
*   Volume dataset $V$.
*   Feature space dimensions (typically 2D: scalar value $v$ and gradient magnitude $\|\nabla v\|$) [S1].
*   Persistence threshold parameter $\tau$ (typically 1% of maximal persistence) [S1].
*   User-defined minimum opacity $\alpha_{min}$ [S1].

**Outputs:**
*   A set of Gaussian Transfer Functions (GTFs), each defined by a mean $\mu$, covariance $\Sigma$, and maximum opacity $\alpha_{max}$, associated with specific valley cells in the feature space [S1].
*   A hierarchical merge tree representing the relationships between these cells [S1].

**Data Structures:**
1.  **2D Density Plot:** A grid representation of the feature space, generated using Continuous Scatterplots (CSPs) [S1]. The grid stores density values for each cell $(i, j)$ [S1].
2.  **Valley Cell:** A connected region in the density plot bounded by "valleys" (local minima) and ridges. Each cell $C_k$ has:
    *   A set of boundary edges.
    *   A persistence value $p_k$, defined as the minimum persistence of its boundaries [S1].
    *   A bounding box in the feature space [S1].
    *   A mean position $\mu_k$ (default: position of maximum density) [S1].
    *   A covariance matrix $\Sigma_k$ derived from the density distribution within the cell [S1].
    *   A size metric $s_k$ (area or volume of the cell in feature space) [S1].
3.  **Merge Tree:** A hierarchical structure where nodes represent valley cells. Edges represent merges. The tree is constructed by iteratively removing boundaries with persistence below a threshold [S1].

**Parameters:**
*   **Persistence Threshold ($\tau$):** Controls simplification. Typically set to 1% of the maximal persistence in the plot. This usually results in fewer than 10 remaining clusters [S1].
*   **Minimum Opacity ($\alpha_{min}$):** User-specified lower bound for opacity [S1].
*   **Mean Position ($\mu$):** Default is the position of the maximum density in the cell. Alternative is the geometric center of the cluster [S1].

**Steps:**

1.  **Generate Density Plot:** Construct a 2D density plot of the volume data features using CSPs [S1].
    *   *Note:* CSPs consider interpolation between data voxels and conservation of physical variables, unlike kernel density estimation which estimates distribution from statistical uncertainty [S1].

2.  **Identify Valley Cells:** Decompose the density plot into valley cells. Each cell represents a potential feature in the volume data [S1].

3.  **Compute Cell Persistence:** For each cell, define its persistence as the smallest persistence of its boundaries. This reflects the degree of separation of the corresponding features [S1].

4.  **Construct Hierarchical Merge Tree:**
    *   Sort all boundaries between adjacent cells by persistence.
    *   Remove boundaries with persistence below the threshold $\tau$ to simplify the tree and remove noisy cells [S1].
    *   The remaining structure forms a hierarchy where merging cells corresponds to removing boundaries [S1].

5.  **Compute Opacity Range ($\alpha_{max}$):** For each node (cell) $i$ in the hierarchy, compute the maximum opacity to mitigate occlusion of small features by larger ones [S1].
    $$ \alpha_i = \alpha_{min} + \frac{W}{g(d_i) \cdot s_i \cdot p_{ci}} $$
    Where:
    *   $s_i$ is the feature size (area of the cell in feature space) [S1].
    *   $p_{ci}$ is the cell persistence of node $i$ [S1].
    *   $g(d_i)$ is a depth-based control function:
        $$ g(d_i) = \frac{d_i - d_{min}}{d_{max} - d_{min}} $$
        where $d_i$ is the depth of node $i$ in the merge tree, and $d_{min}, d_{max}$ are the minimum and maximum depths in the tree [S1].
    *   $W$ is a normalization weight that normalizes $1/(s_i \cdot p_{ci})$ into the range $[0, 1]$ [S1].

6.  **Generate Gaussian Transfer Function (GTF):** For each cell, define a Gaussian opacity function [S1]:
    $$ \alpha(\mathbf{x}) = \alpha_{max} e^{-\frac{1}{2} (\mathbf{x}-\boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1} (\mathbf{x}-\boldsymbol{\mu})} $$
    Where:
    *   $\mathbf{x}$ is a point in the feature space [S1].
    *   $\boldsymbol{\mu}$ is the mean position (default: density maximum) [S1].
    *   $\boldsymbol{\Sigma}^{-1}$ is the inverse of the density covariance matrix in the corresponding valley cell [S1].

7.  **Apply Bounding Box Constraint:** Compute the bounding box for the valley cell. Set $\alpha(\mathbf{x}) = 0$ for any $\mathbf{x}$ outside this bounding box to prevent the Gaussian from contributing to other features [S1].

8.  **User Interaction (Optional):**
    *   Allow manual adjustment of opacity values for each cell [S1].
    *   Allow merging of uninteresting features (e.g., small cells at low densities) [S1].
    *   **Color Assignment:** Map each valley cell to a distinct color using a qualitative color scheme [S1].
        *   When merging cells, color the merged cell with the color of the cell having the larger maximum density [S1].
        *   When splitting cells, restore the original color of the split cell [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_vcc_tf(volume, features, persistence_threshold_pct=0.01, alpha_min=0.0):
    """
    Generates a Valley Cell-Based Clustering Transfer Function.
    
    Args:
        volume: The input volume dataset.
        features: A list of feature names (e.g., ['value', 'gradient_magnitude']).
        persistence_threshold_pct: Percentage of max persistence for simplification.
        alpha_min: Minimum opacity value.
        
    Returns:
        tf_functions: A list of Gaussian Transfer Function objects.
        hierarchy: The merge tree structure.
    """
    # Step 1: Generate 2D Density Plot using CSPs
    # CSP handles interpolation and conservation of physical variables
    density_plot = generate_csp_density_plot(volume, features)
    
    # Step 2: Identify Valley Cells
    # Decompose density plot into connected regions separated by valleys
    valley_cells = decompose_into_valley_cells(density_plot)
    
    # Step 3: Compute Cell Persistence
    # Persistence of a cell is the min persistence of its boundaries
    for cell in valley_cells:
        cell.persistence = min(boundary.persistence for boundary in cell.boundaries)
        
    # Step 4: Construct Hierarchical Merge Tree
    # Sort boundaries by persistence
    all_boundaries = get_all_adjacent_boundaries(valley_cells)
    all_boundaries.sort(key=lambda b: b.persistence)
    
    max_persistence = max(b.persistence for b in all_boundaries) if all_boundaries else 1.0
    threshold = persistence_threshold_pct * max_persistence
    
    # Remove boundaries below threshold to simplify tree
    # This merges cells and builds the hierarchy
    hierarchy = build_merge_tree(valley_cells, all_boundaries, threshold)
    
    # Step 5: Compute Opacity Range for each node in hierarchy
    # Determine depth range for normalization
    depths = [node.depth for node in hierarchy.nodes]
    d_min = min(depths)
    d_max = max(depths)
    
    # Calculate normalization weight W
    # W normalizes 1/(s_i * p_ci) into [0, 1]
    # This requires finding the max value of 1/(s_i * p_ci) across all nodes
    max_inv_size_persistence = max(
        1.0 / (node.size * node.persistence) 
        for node in hierarchy.nodes 
        if node.size > 0 and node.persistence > 0
    )
    W = 1.0 / max_inv_size_persistence if max_inv_size_persistence > 0 else 1.0
    
    for node in hierarchy.nodes:
        # Depth-based control function
        if d_max == d_min:
            g_di = 0.5 # Avoid division by zero, assume middle depth
        else:
            g_di = (node.depth - d_min) / (d_max - d_min)
            
        # Compute alpha_max
        # If size or persistence is zero, handle gracefully (e.g., use small epsilon)
        size = max(node.size, 1e-9)
        persistence = max(node.persistence, 1e-9)
        
        node.alpha_max = alpha_min + W / (g_di * size * persistence)
        
    # Step 6 & 7: Generate Gaussian Transfer Functions
    tf_functions = []
    for node in hierarchy.nodes:
        # Compute covariance matrix from density distribution in the cell
        covariance_matrix = compute_covariance(node.density_data)
        inverse_covariance = invert_matrix(covariance_matrix)
        
        # Determine mean position
        # Default: position of maximum density
        mean_position = find_density_maximum(node.density_data)
        
        # Compute bounding box
        bounding_box = compute_bounding_box(node)
        
        # Create GTF object
        gtf = GaussianTransferFunction(
            alpha_max=node.alpha_max,
            mean=mean_position,
            inverse_covariance=inverse_covariance,
            bounding_box=bounding_box
        )
        tf_functions.append(gtf)
        
    return tf_functions, hierarchy

def evaluate_gtf(gtf, feature_vector):
    """
    Evaluates the opacity for a given feature vector using the GTF.
    """
    # Check if feature vector is within bounding box
    if not is_inside_bounding_box(feature_vector, gtf.bounding_box):
        return 0.0
        
    # Compute Gaussian
    diff = feature_vector - gtf.mean
    exponent = -0.5 * dot_product(diff, multiply_matrix_vector(gtf.inverse_covariance, diff))
    
    opacity = gtf.alpha_max * exp(exponent)
    return opacity
```

## Subtopics

### Hierarchical Feature Representation
A multi-level feature structure created by successive persistence-driven merging of valley cells, enabling interactive exploration through feature splitting, merging, and opacity/color assignment [S1]. Unlike methods requiring a fixed number of clusters or regular widget shapes, this hierarchy adapts to arbitrary feature counts/shapes and incorporates user prior knowledge without manual cluster specification [S1]. The user can interactively merge or split features at any level of the hierarchy, with opacity determined by persistence and size [S1].

### Valley Cell-Based Clustering Transfer Function Generation
The process of assigning colors to valley cells using a qualitative color scheme [S1]. When merging cells in the hierarchy, the merged cell is colored with the color of the cell having the larger maximum density; when splitting, the original color is restored [S1]. This allows distinct visual distinction of features in the feature space [S1].

## Operating characteristics
*   **Time Complexity:** Density plot generation using CSPs is computationally expensive, taking less than 1 hour for a 256x256 resolution with an error threshold of 50 on an Intel Core 2 Duo E6320 1.8 GHz CPU [S1]. Clustering and tree construction are fast, taking approximately 10 seconds [S1].
*   **Memory Complexity:** Depends on the resolution of the density plot and the number of valley cells. The implementation used 2.0 GB RAM and 256 MB VRAM on the reference hardware [S1].
*   **Performance:** Rendering frame rate is approximately 25 fps for fewer than 10 valley cells on an NVIDIA GeForce GTX 260 [S1]. Performance decreases as the number of valley cells increases because the GTF-based volume rendering is evaluated directly on the GPU [S1].
*   **Strengths:** Automatically determines the number of features; uses cell persistence for parameter space simplification; handles arbitrary feature shapes; mitigates occlusion of small features via depth-aware opacity [S1].
*   **Limitations:** Density plot generation is slow; rendering performance scales poorly with a high number of valley cells; requires user interaction for optimal results (merging/splitting) [S1].
*   **When to use:** Suitable for medical data (e.g., CT Teeth, CT facial deformity) and simulation data (e.g., Supernova, Turbulent slip surface) where distinct structures (enamel, dentine, vortex tubes) need to be revealed automatically [S1].
*   **When not to use:** When real-time performance with a large number of features is required, or when the user prefers manual control over binning parameters as in BTF [S1].

## Background
The method was introduced by Y. Wang et al. in 2012 [S1]. It builds on the principle that features tend to form peaks of the density function [S1]. It contrasts with Binning-based Transfer Functions (BTF) [MWCE09] by using topological clustering (Morse complex theory) instead of histogram binning [S1]. The approach uses Continuous Scatterplots (CSPs) [BW08] for density estimation, which considers interpolation between data voxels and conservation of physical variables, unlike kernel density estimation [MWCE09]. The method was implemented using the Cg Language and tested on various volumetric datasets [S1].

## Sources
[S1] Automating Transfer Function Design with Valley Cell‐Based Clustering of 2D Density Plots, https://doi.org/10.1111/j.1467-8659.2012.03122.x, fetched 2026-11-01
